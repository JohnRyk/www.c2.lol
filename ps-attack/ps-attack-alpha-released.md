---
description: After a couple weeks of work I've managed to get PS>Attack in a working state.
---

# PS&gt;Attack Alpha Released

_Originally Published: December 6th, 2015_

#### Well that was easier than expected <a id="well-that-was-easier-than-expected"></a>

The PS&gt;Attack alpha is [out](https://github.com/jaredhaight/psattack/releases) and it does everything I need it to. The most pressing thing that needs to be done is testing and error handling.

#### How it works <a id="how-it-works"></a>

When you start up PS&gt;Attack, the first thing it does is reaches out to the Github API endpoint for PS&gt;Punch and see what the latest release is. It then gets the zip file associated with the release, downloads it and unzips it. The Github API made this super easy to work with but one thing I learned is [you need a user-agent to use the API](https://developer.github.com/v3/#user-agent-required). By default, it appears that the .NET webclient doesn't include a user-agent.

With that figured out, it was relatively easy to map the JSON to a custom object using [JSON.net](http://www.newtonsoft.com/json). This allows us to easily access various attributes of the project. Below is how we download the zip for a release and return the latest release as our custom object.

```text
public static Punch GetPSPunch(Uri URL)
{
    WebClient wc = new System.Net.WebClient();
    // This took a while to figure out: https://developer.github.com/v3/#user-agent-required
    wc.Headers.Add("user-agent", Strings.githubUserAgent);
    string JSON = wc.DownloadString(URL);
    List<Punch> punchList = JsonConvert.DeserializeObject<List<Punch>>(JSON);
    return punchList[0];
}
```

With the zip downloaded and extracted, we clear out the "modules" folder from the source. The modules folder contains the encrypted PowerShell files and the key to decrypt them. We're going to be generating our own, so we don't need that.

To generate the encrypted files we read in a JSON file \("modules.json"\) map it to a custom object like before, iterate through the modules and download the source based off their URL property that was specified in the JSON.

```text
public static List<Module> GetModuleList(string JSON)
{
    List<Module> moduleList = JsonConvert.DeserializeObject<List<Module>>(JSON);
    return moduleList;
}

public static string DownloadFile(string url, string dest)
{
    WebClient wc = new WebClient();
    wc.DownloadFile(url, dest);
    return dest;
}

foreach (Module module in modules)
{
  string dest = Path.Combine(Strings.moduleSrcDir, (module.Name + ".ps1"));
  string encOutfile = punch.modules_dir + module.Name + ".ps1.enc";
  PSAUtils.DownloadFile(module.URL, dest);
  Console.WriteLine("[*] Encrypting: {0}", dest);
  CryptoUtils.EncryptFile(punch, dest, encOutfile);
}
```

The last line of that `foreach` loop passes the downloaded ps1 file to the `EncryptFile` function. `EncryptFile` generates a key, encrypts the file and stores the file and key in the "modules" directory of the downloaded PS&gt;Punch source code.

```text
public static void EncryptFile(Punch punch, string inputFile, string outputFile)
{
    string key = GenerateKey(punch);
    byte[] keyBytes;
    keyBytes = Encoding.Unicode.GetBytes(key);

    Rfc2898DeriveBytes derivedKey = new Rfc2898DeriveBytes(key, keyBytes);

    RijndaelManaged rijndaelCSP = new RijndaelManaged();
    rijndaelCSP.Key = derivedKey.GetBytes(rijndaelCSP.KeySize / 8);
    rijndaelCSP.IV = derivedKey.GetBytes(rijndaelCSP.BlockSize / 8);

    ICryptoTransform encryptor = rijndaelCSP.CreateEncryptor();

    FileStream inputFileStream = new FileStream(inputFile, FileMode.Open, FileAccess.Read);

    byte[] inputFileData = new byte[(int)inputFileStream.Length];
    inputFileStream.Read(inputFileData, 0, (int)inputFileStream.Length);

    FileStream outputFileStream = new FileStream(outputFile, FileMode.Create, FileAccess.Write);

    CryptoStream encryptStream = new CryptoStream(outputFileStream, encryptor, CryptoStreamMode.Write);
    encryptStream.Write(inputFileData, 0, (int)inputFileStream.Length);
    encryptStream.FlushFinalBlock();

    rijndaelCSP.Clear();
    encryptStream.Close();
    inputFileStream.Close();
    outputFileStream.Close();
}
```

Now that we have fresh, encrypted modules in place, the last thing to do is build PS&gt;Punch. In researching this project I learned that the .NET framework actually includes the same compiler thats used by Visual Studio to compile applications \(msbuild.exe\). This means that we can compile PS&gt;Punch anywhere as long as it has the right .NET framework.

```text
public static void BuildPunch(Punch punch)
{
    DateTime now = DateTime.Now;
    string buildDate = String.Format("{0:MMMM dd yyyy} at {0:hh:mm:ss tt}", now);
    using (StreamWriter buildDateFile = new StreamWriter(Path.Combine(punch.res_dir, "attackDate.txt")))
    {
        buildDateFile.Write(buildDate);
    }
    string dotNetDir = System.Runtime.InteropServices.RuntimeEnvironment.GetRuntimeDirectory();
    string msbuildPath = Path.Combine(dotNetDir, "msbuild.exe");
    if (File.Exists(msbuildPath))
    {
        Process msbuild = new Process();
        msbuild.StartInfo.FileName = msbuildPath;
        msbuild.StartInfo.Arguments = punch.build_args;
        msbuild.StartInfo.UseShellExecute = false;
        msbuild.StartInfo.RedirectStandardOutput = true;
        msbuild.StartInfo.RedirectStandardError = true;

        Console.WriteLine("Running build with this command: {0} {1}", msbuild.StartInfo.FileName, msbuild.StartInfo.Arguments);

        msbuild.Start();
        string output = msbuild.StandardOutput.ReadToEnd();
        Console.WriteLine(output);
        string err = msbuild.StandardError.ReadToEnd();
        Console.WriteLine(err);
        msbuild.WaitForExit();
        msbuild.Close();
    }
}
```

As long as the computer has .NET 3.5, it _should_ compile just fine.

#### Demo <a id="demo"></a>

So putting it all together we end up with [this](https://www.youtube.com/watch?v=i-RvUahn4ls).

The alpha for PS&gt;Attack is [available here](https://github.com/jaredhaight/psattack/releases). Give it a shot and let me know how it works for you. You can either submit an [issue](https://github.com/jaredhaight/psattack/issues) or by emailing me at jaredhaight `at` protonmail.com.

