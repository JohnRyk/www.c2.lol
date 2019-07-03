---
description: >-
  Recently I was working on getting a meterpreter session up and running, but
  the parent PHP process kept getting killed off. Here is what I learned to get
  around that.
---

# Starting a Process under systemd

_Originally Published: April 15th, 2015_

I was working on getting shell on a Linux server recently. It was a pretty typical LAMP stack and hadn't been updated in a while. In digging around, I found that it was running an old version of "Advanced Comment System" that was vulnerable to remote code inclusion. Running phpinfo\(\); told me that the server was running an old linux kernel that was vulnerable to a privilege escalation attack.

I then took a public exploit and modified it so that instead of escalating priviliges and running /bin/sh locally, it would run a reverse meterperter shell that I'd generated. Using the remote file inclusion vulnerability, I ran some PHP that downloaded the payloads and than ran them on the server.

Everything worked great except that the PHP process that was executing the shell kept getting killed off, dropping my access.

I googled around and found [this question on superuser](http://superuser.com/questions/178587/how-do-i-detach-a-process-from-terminal-entirely). Down the page someone suggests piping the process to "at". Doing this schedules the program to run immediately and will run it under initd \(or systemd\).So the final command ended up looking something like this:

```text
/tmp/exploit.bin | at
```

What a clever way to address this problem. Definitely something I'll keep in mind for the future.

