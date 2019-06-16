# OS X computer certificate enrollment in Microsoft CAs

_Originally Published: Janurary 19th, 2015_

We recently had a need to get our Macs to request certs from our AD certificate authorities. This is _mainly_ covered [here](http://support.apple.com/en-us/HT5357), but here are some hang ups that myself and others seemed to have run into:

#### Some basics <a id="some-basics"></a>

* Make sure the Mac is bound to AD
* Make sure that the "Domain Computers" group has permissions to the cert template that you're using.

#### Getting a computer cert <a id="getting-a-computer-cert"></a>

Using the above steps, I was able to get a cert for my user account, but not for the computer. What ended up getting me was that my user account had a kerberos ticket, but the Mac did not. You can verify this by running:

```text
klist -l
```

If you do not see your computer name listed, run this to get a ticket \(where HOSTNAME is your hostname\):

```text
kinit -k HOSTNAME$
```

For example if your hostname was "test-computer", you'd run:

```text
kinit -k test-computer$
```

Running klist again, you should see your computers kerberos ticket listed and your cert request should succeed.

