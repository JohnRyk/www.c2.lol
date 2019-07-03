---
description: >-
  In trying to troubleshoot why a payload wasn't executing properly I had to
  learn some C.
---

# Printing out hex with C

_Originally Published: March 14th, 2015_

I am working my way through the [OSCP](https://www.offensive-security.com/information-security-certifications/oscp-offensive-security-certified-professional/) and this means getting a crash course in C and to a lesser extent assembly. In troubleshooting why a certain buffer overflow exploit wasn't executing properly on the remote computer, I had to work out how to print the payload so I could see the final product before it was sent. The issue was that anything I printed was being output to ASCII instead of raw hex for example "\x43" was printing out as "C".

There was A LOT of learning involved in trying to get this figured out. Probably the biggest lesson was that what I'm used to as strings in Python are arrays of characters in C. Once I'd learned that it became a matter of writing a loop that itterated through the array and printed out each character as hex. The final result was this function.

```text
void print_hex(unsigned char hexarray[3550]) {
    int i;
    printf("\n\n***** Printing Hex ****\n\n");
    for (i=0; i<3550; i++)
    {
        printf("0x%02X",hexarray[i]);
    }
    printf("\n\n\n");
}
```

It is super basic, but it helped me get done what I needed to.

