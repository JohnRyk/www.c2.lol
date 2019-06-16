---
description: >-
  PS>Attack is my first big C# project, my first big open source project and a
  way to give back to the infosec community (and learn some in-depth stuff about
  PowerShell).
---

# PS&gt;Attack v.000001

_Originally Published: December 2nd, 2015_

#### What the hell is PS&gt;Attack? <a id="what-the-hell-is-ps-attack-"></a>

[PS&gt;Attack](https://www.github.com/jaredhaight/PSAttack/) is the offensive PowerShell tool that I'm working on. The goal is to make it ridiculously easy for pentesters to get started with using PowerShell on engagements. It takes a lot of the best tools for attacking with PowerShell, encrypts them and stores them in a self contained exe. This exe decrypts the files straight into memory and recreates a PowerShell environment without calling PowerShell.exe, making it harder to detect by Antivirus and Incident Response teams. Assuming my talk gets accepted, it'll be released at [CarolinaCon](http://carolinacon.org) in 2016.

#### Getting started <a id="getting-started"></a>

So I've started working in earnest on PS&gt;Attack, my offensive PowerShell tool \(and the namesake for this site\). So far things have gone surprisingly well. I'm about 12 to 16 hours in and I've been focused primarily on the getting a portable, custom PowerShell environment \([PS&gt;Punch](https://github.com/jaredhaight/PSPunch/)\) put together that does what I want it to. So far I have the following things straight:

**Decrypting Files**

Files can be stored in the executable as embeddable resources and then decrypted into memory. This works by taking a ModuleStream object \(the embedded file\) and sending decrypting it to a MemoryStream object. The code right now looks like this:

```text
        public static MemoryStream DecryptFile(Stream inputStream, string key)
        {
            byte[] keyBytes = Encoding.Unicode.GetBytes(key);

            Rfc2898DeriveBytes derivedKey = new Rfc2898DeriveBytes(key, keyBytes);

            RijndaelManaged rijndaelCSP = new RijndaelManaged();
            rijndaelCSP.Key = derivedKey.GetBytes(rijndaelCSP.KeySize / 8);
            rijndaelCSP.IV = derivedKey.GetBytes(rijndaelCSP.BlockSize / 8);
            ICryptoTransform decryptor = rijndaelCSP.CreateDecryptor();

            CryptoStream decryptStream = new CryptoStream(inputStream, decryptor, CryptoStreamMode.Read);
            byte[] inputFileData = new byte[(int)inputStream.Length];
            string contents = new StreamReader(decryptStream).ReadToEnd();
            byte[] unicodes = Encoding.Unicode.GetBytes(contents);

            MemoryStream outputMemoryStream = new MemoryStream(unicodes);
            rijndaelCSP.Clear();

            decryptStream.Close();
            inputStream.Close();
            return outputMemoryStream;
        }
```

In this example, `inputStream` is the ModuleStream and `key` is the string used to encrypt the files. The key is also retrieve from an embedded file that will be setup when the client is built.

**Executing PowerShell without PowerShell.exe**

Like most things in this project, there are probably much better ways to handle this. It works though and right now that's the important part. Everything involved in this process was cribbed from [PoshSecFramework](https://github.com/PoshSec/PoshSecFramework) as well as a ton of googling, MSDN articles and StackOverflow questions. There are quick ways to execute PowerShell from C\#, [AwesomerShell](https://github.com/Ben0xA/AwesomerShell) by Ben0xA is a great example of this. AwesomerShell basically boils down to this:

```text
        static void Main(string[] args)
        {
            Console.Write("aps> ");
            String cmd = Console.ReadLine();

            Runspace rs = RunspaceFactory.CreateRunspace();
            rs.Open();

            PowerShell ps = PowerShell.Create();
            ps.Runspace = rs;
            ps.AddScript(cmd);

            Collection<PSObject> output = ps.Invoke();
            if (output != null)
            {
                foreach (PSObject rtnItem in output)
                {
                    Console.WriteLine(rtnItem.ToString());
                }
            }
            rs.Close();
            Console.Write("Press any key to exit.");
            Console.ReadLine();
        }
```

* You write out a prompt \(in this case "aps&gt; "\)
* Read input from the user.
* Create a runspace and create a PowerShell object
* Pass the input to the PowerShell object.
* Invoke the PowerShell object.
* Take the output and write it to screen.

To do what I want to do though \(right now tab-completion\), I needed to get more in-depth and create a custom PowerShell host. A host is something that allows you to interact with a PowerShell runspace. A PowerShell runspace is the actual environment that executes PowerShell code. PowerShell.exe and PowerShell ISE are examples of hosts. They don't execute PowerShell directly, rather they let you interact with a runspace that executes PowerShell.

To create a PowerShell host, I needed to specify three things:

* A PowerShell Raw UI _\(this might actually be optional\)_
* A PowerShell UI
* A PowerShell Host

Each of these depend on the last.

To put this together I cribbed code from [PoshSecFramework](https://github.com/PoshSec/PoshSecFramework) and [this post](http://community.bartdesmet.net/blogs/bart/archive/2008/07/06/windows-PowerShell-through-ironruby-writing-a-custom-pshost.aspx) about integrating PowerShell and Ruby.

The relevant code for PS&gt;Punch is [available here](https://github.com/jaredhaight/PSPunch/tree/master/PSPunch/PSPunchShell). It's rapidly changing right now, but hopefully it will be helpful to people trying to accomplish similar tasks. I'll also probably write up another blog entry later that goes more in-depth about how these components interact.

#### Recreating a Console <a id="recreating-a-console"></a>

Because I want to act on certain key presses, I can't simply use `Console.ReadLine()` to get my input, I have to use `Console.ReadKey()` instead and check every character as it's typed.

If the character typed is not a tab, backspace or enter: I take that character, append it to a list of the other characters typed, clear the console line \(that's important later\) and then print out the entire command string to console.

If the character is `backspace`, I remove the last character in the command string, clear the line and reprint the new command string.

If the character is `enter`, I send the command string off to be executed.

Tab Completion works in a Proof of Concept standpoint right now. If a `tab` is entered, I wrap the current command string in `get-command` and `*` and then send it off to be executed by PowerShell. My next task is to actually cycle through commands, which shouldn't be too hard to get set up.

Currently this all looks like this \(I'm using a custom `PunchInput` object to pass details about the console between the UI and execution elements of program\):

```text
        // Main from PSPunch/Program.cs
        static void Main(string[] args)
        {
            string prompt = "PSPUNCH! #> ";
            // Code removed for clarity
            while (true)
            {
                // Code removed for clarity
                while (punchInput.keyInfo.Key != ConsoleKey.Enter)
                {
                    punchInput = Processing.CommandProcessor(punchInput);
                    if (punchInput.output != null)
                    {
                        Console.WriteLine("\n" + punchInput.output);
                        consoleTopPos = Console.CursorTop;
                        Console.ForegroundColor = ConsoleColor.White;
                        Console.Write(prompt);
                        Console.ForegroundColor = ConsoleColor.Yellow;
                    }
                    Console.SetCursorPosition(prompt.Length, consoleTopPos);
                    Console.Write(new string(' ', Console.WindowWidth));
                    Console.SetCursorPosition(prompt.Length, consoleTopPos);
                    Console.Write(punchInput.cmd);
                    punchInput.keyInfo = Console.ReadKey();
                }
            }
        }
        // CommandProcessor pulled from PSPunch/PSPunchInput/Processing.cs
        public static PunchInput CommandProcessor(PunchInput punchInput)
        {
            punchInput.output = null;
            if (punchInput.keyInfo.Key == ConsoleKey.Backspace)
            {
                if (punchInput.cmd.Length > 0)
                {
                    punchInput.cmd = punchInput.cmd.Remove(punchInput.cmd.Length - 1);
                }
            }
            else if (punchInput.keyInfo.Key == ConsoleKey.Tab)
            {
                string origInput = punchInput.cmd;
                punchInput.cmd = "Get-Command " + punchInput.cmd + "*";
                punchInput = PSExec(punchInput);
                punchInput.cmd = origInput;
            }
            else
            {
                punchInput.cmd += punchInput.keyInfo.KeyChar;
            }
            return punchInput;
        }
```

It needs to be cleaned up, but it works..

#### Where it's at now <a id="where-it-s-at-now"></a>

So far things are going really well. After only 3 days of works I can call PS&gt;Punch, files are decrypted into memory, typing works, command execution works and my tab-completion PoC works. I'm realy optimistic about where things are headed.[ Here is a quick demo of the project as it stands right now](https://www.youtube.com/watch?v=S5kD5dSyVwc).

