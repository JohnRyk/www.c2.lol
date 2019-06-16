---
description: >-
  Chris Long recently released DetectionLab, a Vagrant project that allows
  defenders or attackers to quickly build an Active Directory domain configured
  with security monitoring tooling and logging best
---

# Setting up Chris Long's DetectionLab

_Originally Published:  December 18th, 2017_

I'm super stoked about this project. It's incredibly important for an attacker to be aware of their IOCs, but there's quite a bit of knowledge required to build out security monitoring. This project handles:

* Deploying Splunk
* Configuring WEF
* Configuring Sysmon
* Configuring PowerShell transcription logging
* Deploying osquery
* Deploying Fleet

All of this is deployed to current best practices, allowing an attacker \(or defender\) to quickly have a lab where they can run through attacks and see what sort of tracks that they're leaving behind.

### A crash course in DevOps <a id="a-crash-course-in-devops"></a>

Playing with DetectionLab was a great excuse to mess around with some devopsy tools that I've been meaning to look at for awhile.

* [Vagrant](https://www.vagrantup.com) was designed to simplify deploying development environments. You declare your environment in a config file, your developers download this file and then run `vagrant up`, which handles building out VMs and configuring them. Vagrant is able to deploy VMs to many different providers \(AWS, VMWare, Hyper-V, Virtualbox, etc\), but the config file has to be written to support each provider. DetectionLab supports VMWare and Virtualbox. It's worth noting that the VMWare plugin for Vagrant costs $80 \(at the time of this writing\), the Virtualbox plug is free and installed automatically.
* [Packer](https://www.packer.io/) compliments Vagrant by building gold images. Like Vagrant, it supports multiple providers including its own 'box' format, which is what Vagrant leverages. In DetectionLab, Packer handles downloading Windows ISOs, installing Windows, patching, and installing some custom software.

The README for DetectionLab does a good job of walking through setting all of this up. The basic process though is:

1. Install Vagrant
2. Install Packer
3. Run Packer to build the Windows images
4. Run Vagrant to build the lab

An interesting approach to this lab \(which seems like it might be convention\) is that it leverages the hypervisors port forwarding for access to resources. In labs that I build, I typically have a shared network between my host and the VMs and just access VM resources directly.

### Customizing <a id="customizing"></a>

Most of the customization that you would want to do would be handled by editing the [Vagrantfile](https://github.com/clong/DetectionLab/blob/master/Vagrant/Vagrantfile), which is pretty straightforward.

In my case, I ended up customizing the Packer JSON files to point to local ISOs instead of a URL. This offered a couple of advantages for me:

1. It allowed me to use my MSDN ISOs
2. It meant that every time I built the Packer image, it didn't have to redownload the ISOs

Customizing this was just a matter of pointing the 'iso\_url' attribute in each of the [Windows JSON files to a local ISO](https://github.com/jaredhaight/DetectionLab/blob/master/Packer/windows_10.json#L173) and updating the hash. To get the hash, I used the magic of PowerShell.

```text
PS> Get-FileHash -Path D:\ISOs\en_windows_10_enterprise_version_1703_updated_june_2017_x64_dvd_10720664.iso -Algorithm SHA1 | Format-List

Algorithm : SHA1
Hash      : D7EF32D857D444886D941A49C39E76D60ACA4485
Path      : D:\ISOs\en_windows_10_enterprise_version_1703_updated_june_2017_x64_dvd_10720664.iso
```

### Issues <a id="issues"></a>

I'll be honest, it took me like 3 days to get this successfully deployed. All of my issues could have been avoided had I read the README more carefully. Go read the README. Things I learned the hard way:

1. You can only build one Packer image at a time. It will take about an hour per image.
2. The README has an [issues](https://github.com/clong/DetectionLab/blob/master/README.md#known-issues-and-workarounds) section which covered the other issues I ran into. \(Namely, the port forwarding issue\)

Ultimately I was able to successfully deploy the lab by running `vagrant up`, which failed on the port forwarding. I then ran `vagrant reload dc --provision` and then deployed the remaining machines with `vagrant up wef` and `vagrant up win10`

The port forwarding issue seems to be a bug with Vagrant, otherwise the lab would have deployed successfully I think.

### Going Forward <a id="going-forward"></a>

I can't wait to spend some more time in this lab. I'll probably end up writing some more stuff as I learn how to use Fleet and Splunk a bit. Major thanks to Chris for all the effort that went into putting this together.

