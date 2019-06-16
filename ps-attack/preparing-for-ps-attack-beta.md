---
description: >-
  A lot of work has gone into PS>Attack over the last month and almost all of
  the features work. With any luck, I'll get the last checkbox checked this
  weekend and it can move into beta.
---

# Preparing for PS&gt;Attack Beta

_Originally Published: Janurary 9th, 2016_

The past month PS&gt;Attack has really taken shape. Error handling has been added to a lot of aspects of the project, auto-complete works in the PS&gt;Punch console and more modules have been added. PS&gt;Punch is becoming a console I can actually feel comfortable with. This weekend I'll begin working on the `get-attack` module which will serve as an attack search engine for the console, making it easier for people new to offensive PowerShell to find the attack they're looking for. Once that's done, we can enter a beta state and start fixing bugs.

So much work has gone into PS&gt;Attack the last month that it's easy to lose track, so I wanted to write an update on what's new and improved.

#### General maintenance <a id="general-maintenance"></a>

* Messages, banners, prompts, etc are in their own file now. This makes it easy to update verbiage in the programs.
* Colors are in their own file now too \(see reasoning above\)
* Printing to console is much improved and behaves like a native console window. This was achieved by changing the various "Write" functions in the PowerShellHostUI to print to console directly. In hindsight, it was dumb to do this any other way.

```text
// snippet from PSPunchHostUserInterface.cs
public override void Write(string message)
{
    Console.ForegroundColor = PSColors.outputText;
    Console.Write(message);
}

public override void WriteDebugLine(string message)
{
    Console.ForegroundColor = PSColors.debugText;
    Console.WriteLine("DEBUG: {0}", message);
    Console.ForegroundColor = PSColors.outputText;
}
```

* [inveigh](https://github.com/Kevin-Robertson/Inveigh) and [powercat](https://github.com/besimorhino/powercat) have been added.
* Testing has been done against ~80% of the commands included.
* Clean error handling has been added to many aspects of the projects. It's now actually kind of rare for me to find a crashing bug.

#### Encrypted Embedded File Names <a id="encrypted-embedded-file-names"></a>

The C\# solution file is used to declare how a project should be build. Ths includes what files should be embedded into the final executable. For PS&gt;Punch, we use this to include the various encrypted payloads in the compiled executable.

The problem that came up with this is that we need to know the file names that we're going to be using before we start building. Intially, PS&gt;Punch was hard coded to look for specific files like `invoke-mimikatz.ps1.enc`. This meant that when PS&gt;Attack downloads `invoke-mimikatz.ps1`, we encrypt that and we have to call it `invoke-mimikatz.ps1.enc` so that when we build PS&gt;Punch, it gets included.

The problem with this is it makes it a lot easier for antivirus to catch on to what we're doing. It's a static signature and we're using words like "mimikatz" which is a dead give away that we're up to no good.

Now PS&gt;Attack encrypts the filenames with the same unique key that it uses to encrypt the rest of the file. It then rebuilds the solution file for PS&gt;Punch to include references to the new filenames. This was done with [C\# text templates](https://msdn.microsoft.com/en-us/library/ee844259.aspx).

When PS&gt;Attack goes to build PS&gt;Punch, it calls `BuildCsproj`, which creates a list of encrypted filenames and passes that to a `PSPunchCSProj` object. This object was automatically created through the text template process.

```text
public static void BuildCsproj(List<Module> modules, Punch punch)
{
    punch.ClearCsproj();
    List<string> files = new List<string>();
    foreach (Module module in modules)
    {
        files.Add(CryptoUtils.EncryptString(punch, module.Name));
    }
    PSPunchCSProj csproj = new PSPunchCSProj();
    csproj.Session = new Dictionary<string, object>();
    csproj.Session.Add("files", files);
    csproj.Initialize();

    var generatedCode = csproj.TransformText();
    Console.WriteLine("Writing PSPunch.csproj to {0}", punch.csproj_file);
    File.WriteAllText(punch.csproj_file, generatedCode);
}
```

When the `PSPunchCSProj` object is initialized, it takes that list and feeds it through this section of the template

```text
  <ItemGroup>
    <None Include="app.config" />
    <# foreach (string file in this.files ) {
    WriteLine("<EmbeddedResource Include=\"Modules\\{0}.ps1.enc\"/>", file);
     } #>
  </ItemGroup>
```

We then generate the new project file \(with `TransformText()`\) and save that into the PS&gt;Punch build directory.

#### Get Credentials <a id="get-credentials"></a>

I wrote a simple implementation of `Get-Credentials` as it was required by some of the modules. This is really straight forward simply prompting for a username and password, then passing those back as `PSCredential` object. Some trickery is used to display an asterisk when the password is being typed in \(instead of displaying the password as it's typed\) and to convert the entered password to a `SecureString`.

```text
// Function used for PromptForCredential
private PSCredential GetCreds(string caption, string message)
{
    Console.WriteLine(caption);
    Console.WriteLine(message);
    Console.Write("Enter Username (domain\\user): ");
    string userName = Console.ReadLine();
    Console.Write("Enter Pass: ");
    ConsoleKeyInfo info = Console.ReadKey(true);
    string password = "";
    while (info.Key != ConsoleKey.Enter)
    {
        if (info.Key != ConsoleKey.Backspace)
        {
            Console.Write("*");
            password += info.KeyChar;
        }
        else if (info.Key == ConsoleKey.Backspace)
        {
            if (!string.IsNullOrEmpty(password))
            {
                password = password.Substring(0, password.Length - 1);
                int pos = Console.CursorLeft;
                Console.SetCursorPosition(pos - 1, Console.CursorTop);
                Console.Write(" ");
                Console.SetCursorPosition(pos - 1, Console.CursorTop);
            }
        }
        info = Console.ReadKey(true);
    }

    SecureString secPasswd = new SecureString();
    foreach (char c in password)
    {
        secPasswd.AppendChar(c);
    }
    secPasswd.MakeReadOnly();
    return new PSCredential(userName, secPasswd);
}
```

#### Autocomplete <a id="autocomplete"></a>

Any good console needs tab completion so a **lot** of time has been spent making sure that autocomplete in PS&gt;Punch is solid. PowerShell offers great tab-expansion, but the way it's implemented changed between PowerShell v2 and v3. One of the design descions for PS&gt;Punch is that it needs to work on an unpatched Windows 7 box \(PowerShell v2\) through to an updated Windows 10 box \(PowerShell v5\).

So I had two options in implementing tab completion in PS&gt;Punch: I could write code to detect the version of PowerShell that's available and route autocomplete requests to the right handler that would handle calling native tab-expansion or I could implement my own tab completion. I opted for the latter, partially for consistency in behavior across versions of PS and partially because I'd already had a decent implemention when I found out about how I could call native PowerShell tab-expansion.

Over the last month autocomplete has become much more efficient and useful. Improvments include:

* Being able to autocomplete properly given a complex string.
* Completion of command parameters.
* Completion of variables

**Analyzing how autocomplete works**

A common convention in PowerShell is piping the output from one command to another, for example: `Get-NetAdapter | Out-File -FilePath C:\netinfo.txt -Append`. In that example, there's a lot that we could be working with: multiple commands, multiple parameters and a filename. So for autocomplete to be useful, we need to be able to take everything typed after the prompt, analyze it and figure out the context of what the user is trying to complete.

The code below is run to chop up what we're autocompleting. In the end we get a `displayCmdSeed` and an `autocompleteSeed`. We'll use the `autocompleteSeed` to determine what the best course of action for autocompletion is. The `displayCmdSeed` is used to figure out what command we're working with.

```text
int lastSpace = punchState.displayCmd.LastIndexOf(" ");
if (lastSpace > 0)
{
    // get the command that we're autocompleting for by looking for the last space and pipe
    // anything after the last space we're going to try and autocomplete. Anything between the
    // last pipe and last space we assume is a command. 
    int lastPipe = punchState.displayCmd.Substring(0, lastSpace + 1).LastIndexOf("|");
    punchState.autocompleteSeed = punchState.displayCmd.Substring(lastSpace);
    if (lastSpace - lastPipe > 2)
    {
        punchState.displayCmdSeed = punchState.displayCmd.Substring(lastPipe + 1, (lastSpace - lastPipe));
    }
    else
    {
        punchState.displayCmdSeed = punchState.displayCmd.Substring(0, lastSpace);
    }
    // trim leading space from command in the event of "cmd | cmd"
    if (punchState.displayCmdSeed.IndexOf(" ").Equals(0))
    {
        punchState.displayCmdSeed = punchState.displayCmd.Substring(1, lastSpace);
    }
}
else
{
    punchState.autocompleteSeed = punchState.displayCmd;
    punchState.displayCmdSeed = "";
}
if (punchState.autocompleteSeed.Length == 0)
{
    return punchState;
}
```

Now that we know what we're trying to autocomplete \(our `autocompleteSeed`\), we can route it to the appropriate handler.

```text
// route to appropriate autcomplete handler
if (punchState.autocompleteSeed.Contains(" -"))
{
    punchState = paramAutoComplete(punchState);
}
else if (punchState.autocompleteSeed.Contains("$"))
{
    punchState = variableAutoComplete(punchState);
}
else if (punchState.autocompleteSeed.Contains(":") || punchState.autocompleteSeed.Contains("\\"))
{
    punchState = pathAutoComplete(punchState);
}
else
{
    punchState = cmdAutoComplete(punchState);
}
```

I won't show off every handler here, but the parameter handler is kind of cool. In the case of autocompleting a parameter we need to know what snippet of a parameter has been typed and what command it's associated with. To do this we're going to chop up the `displayCmdSeed`.

First we figure out the parameter that we're trying to autocomplete by looking for the characters after the last dash, we assign this to paramSeed. We then take everything in `displayCmdSeed` up to the first space and assign that to `paramCmd`.

We put this together into the following PowerShell command

`(Get-Command paramCmd).Parameters.Keys | Where {$_ -like paramSeed*}`

We run that command and then grab the results: a list of parameters for the command that match our `paramSeed`.

```text
static PunchState paramAutoComplete(PunchState punchState)
{
    punchState.loopType = "param";
    int lastParam = punchState.displayCmd.LastIndexOf(" -");
    string paramSeed = punchState.displayCmd.Substring(lastParam).Replace(" -", "");
    int firstSpace = punchState.displayCmd.IndexOf(" ");
    string paramCmd = punchState.displayCmdSeed.Substring(0, firstSpace);
    punchState.cmd = "(Get-Command " + paramCmd + ").Parameters.Keys | Where{$_ -like '" + paramSeed + "*'}";
    punchState = Processing.PSExec(punchState);
    return punchState;
}
```

With a list of parameters, it's pretty easy to just cycle through those each time the user hits tab.

#### Beta and beyond <a id="beta-and-beyond"></a>

The beta for PS&gt;Attack should be out in the next couple of weeks. After that development will switch over to bug fixes. With the release of the beta I'll also be actively encouraging people to submit issues and pull requests.

You can follow along with development of PS&gt;Attack [on Github](https://github.com/jaredhaight/PSAttack).

