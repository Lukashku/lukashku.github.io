---
layout: post
title: HackTheBox ForwardSlash (10.10.10.183) Writeup
description: >
  Hackthebox ForwardSlash Writeup
image: /assets/img/forwardslash.png
noindex: false
---

This box is still active therefore the writeup is protected.

You can download this writeup by [clicking here](/active/pdf/forwardslash1.pdf)

The password is the root hash. Example:
```
root:$6$JNnF8uyhyLU8TRvi$START--->cbNcEL/fX4taIJdlEp.qlx9Nddfx.xrEZi.FxiH4v6UwoioTdSPhIu8uLVPWFXAVwVElGfsd46Dpg4zdOxfd0<---STOP:18365:0:99999:7:::
```
Alternatively, you can download the writeup by  [clicking here](/active/pdf/forwardslash2.pdf)

The password is the md5 hash of the entire root users shadow file entree. Example:
```
user@pc1:$ echo -n 'root:$6$JNnF8uyhyLU8TRvi$cbNcEL/fX4taIJdlEp.qlx9Nddfx.xrEZi.FxiH4v6UwoioTdSPhIu8uLVPWFXAVwVElGffi1sFDpg4zdOxfd0:18365:0:99999:7:::' | md5sum
e1ab7cc1abe611dd70fd96e5628ecc05  -
```

