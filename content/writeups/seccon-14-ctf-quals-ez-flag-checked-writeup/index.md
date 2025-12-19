+++
title = 'SECCON 14 CTF Quals Writeup: ez_flag_checker'
description = 'Writeup for SECCON 14 CTF Quals Rev Eng Challenge ez-flag-checker'
date = 2025-12-19
author = 'Shruti Priya'
+++

**Author: Shruti Priya**

Hello again and welcome to the writeup for [SECCON 14 CTF Quals Reverse Engineer challenge: ez_flag_checker](https://score.ctf.seccon.jp/challenges?#Ez%20Flag%20Checker-20). This Reverse Engineering challenge required us to decrypt the flag hidden behind an encryption function in the binary. 

## Understanding the binary

Let's start with the simplest way, running the binary and analysing the output. This challenge binary simply asks for the flag when executed.

```bash
$ ./chall
Enter flag: 
```
Let's enter something random and see how the binary reacts. The flag format for this CTF begins with `SECCON{` and ends with another `}` so let's follow the format.

```bash
$ ./chall
Enter flag: SECCON{hello}
wrong :(
```

The binary reacts with a string `wrong :(`. I'd wager the challenge takes the actual flag and checks it against something in the memory. If this check fails, we get the output `wrong :(`. Now, let's disassemble the binary and analyse its functionality.

## Disassembling the binary

Disassembling the binary in IDA, I notice that at one point the challenge compares the user's input with the string `SECCON{`.

```nasm
loc_132C:
lea     rax, [rbp+buf]
mov     edx, 7          ; n
lea     rcx, s2         ; "SECCON{"
mov     rsi, rcx        ; s2
mov     rdi, rax        ; s1
call    cs:strncmp_ptr
test    eax, eax
jnz     short loc_137A
```

The `strncmp_ptr` compares the first 7 characters of the user's input with the string in memory. Therefore, I was correct for entering the flag format. After checking these first 7 characters, the challenge continues to call another function called `sigma_encrypt`.

```nasm
loc_1394:
lea     rax, [rbp+buf]
add     rax, 7
mov     [rbp+message], rax
lea     rcx, [rbp+user_tag]
mov     rax, [rbp+message]
mov     edx, 12h        ; len
mov     rsi, rcx        ; out
mov     rdi, rax        ; message
call    sigma_encrypt
lea     rax, [rbp+user_tag]
mov     edx, 12h        ; n
lea     rcx, flag_enc
mov     rsi, rcx        ; s2
mov     rdi, rax        ; s1
call    cs:memcmp_ptr
test    eax, eax
setz    al
movzx   eax, al
mov     [rbp+ok], eax
cmp     [rbp+ok], 0
jz      short loc_1411
```

Notice also the `flag_enc` memory pointer. This is the encrypted version of the flag. The challenge takes the user input, compares the first 7 characters with `SECCON{`. If this check passes, the challenge calls the `sigma_encrypt` function to encrypt the user input. Then the challenge compares this output with the encrypted flag stored in memory. 

Since I can access both the `sigma_encrypt` function and `flag_enc`, the intended pathway for solving this challenge must involve decrypting the flag.

## Sigma Encrypt

The `sigma_encrypt` function takes 3 arguments: `message`, `out`, `len`. This function also has a curious array called `sigma_words`. Looking at the data in `sigma_words`, it contains 4 double words (1 double word is 4-bytes).

```c
void __cdecl sigma_encrypt(const char *message, uint8_t *out, size_t len)
{
  int i; // [rsp+30h] [rbp-30h]
  uint32_t w; // [rsp+34h] [rbp-2Ch]
  size_t i_0; // [rsp+38h] [rbp-28h]
  uint8_t key_bytes[24]; // [rsp+40h] [rbp-20h]
  unsigned __int64 v7; // [rsp+58h] [rbp-8h]

  v7 = __readfsqword(0x28u);
  for ( i = 0; i <= 3; ++i )
  {
    w = sigma_words[i];
    key_bytes[4 * i] = w;
    key_bytes[4 * i + 1] = BYTE1(w);
    key_bytes[4 * i + 2] = BYTE2(w);
    key_bytes[4 * i + 3] = HIBYTE(w);
  }
  for ( i_0 = 0; i_0 < len; ++i_0 )
    out[i_0] = (i_0 + key_bytes[i_0 & 0xF]) ^ message[i_0];
  if ( v7 != __readfsqword(0x28u) )
    _stack_chk_fail();
}
```

The encryption function iterates over the `sigma_words` array and adds certain bytes to the `key_bytes` array. Then, the function iterates over the `key_bytes`, performs an AND operation between the `key_byte` and `0xF` and XOR-s the result with the `message` byte.

Since the encryption performs an XOR operation, we can get the real flag by just plugging in the encrypted flag bytes instead of the `message` in the `sigma_encrypt` function.

## Crafting the exploit

I create a Python implementation of the `sigma_encrypt` function and extract the flag-bytes from the `flag_enc` memory pointer.

```python
sigma_words = [0x61707865, 0x3320646E, 0x79622D32, 0x6B206574]

# Function to derive key_bytes from sigma_words
def derive_key_bytes():
    key_bytes = []
    for i in range(4): 
        w = sigma_words[i]
        key_bytes.append(w & 0xFF)          # Low byte
        key_bytes.append((w >> 8) & 0xFF)   # 2nd byte
        key_bytes.append((w >> 16) & 0xFF)  # 3rd byte
        key_bytes.append((w >> 24) & 0xFF)  # High byte
    return key_bytes

# Decryption function
def sigma_decrypt(encrypted_message, key_bytes):
    decrypted_message = bytearray(len(encrypted_message))
    for i in range(len(encrypted_message)):
        decrypted_message[i] = encrypted_message[i] ^ (i + key_bytes[i & 0xF])
    return decrypted_message


# flag_enc instead of message
encrypted_message = bytearray([0x03,0x15,0x13,0x03,0x11,0x55,0x1f,0x43,0x63,0x61,0x59,0xef,0xbc,0x10,0x1f,0x43,0x54,0xa8])

key_bytes = derive_key_bytes()

decrypted_message = sigma_decrypt(encrypted_message, key_bytes)

print("Decrypted message:", decrypted_message.decode())

```

Let's execute this and get our flag!

```bash
$ python ez-flag-exploit.py
Decrypted message: flagc<9yYW5k<b19!!
$ ./chall
Enter flag: SECCON{flagc<9yYW5k<b19!!}
wrong :(
$ 
```

Unfortunately, this did not work. For some reason, the challenge does not accept our flag. Let's debug this in `pwndbg` and see what is going wrong.

## Debugging the exploit

```bash
$ pwndbg ./chall
pwndbg> b *main+240
Breakpoint 1 at 0x1141: file main.c, line 9.
pwndbg> r
Starting program: ~/ctfs/seccon/ez_flag_checker/chall 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Enter flag: SECCON{flagc<9yYW5k<b19!!}
```

Lots of things happening here! I am running this binary from within `pwndbg`. I add a breakpoint at the instruction where the challenge compares the user input with the encrypted flag in memory. Then I run the binary and enter the flag.

```gdb
RDI  0x7fffffffd890 ◂— 0x431f5b1103131503
RSI  0x555555556010 (flag_enc) ◂— 0x431f551103131503
```

Checking the registers right before the strings are compared, I notice that there is a slight difference in my flag and the encrypted flag. Notice how the 3rd byte is `0x5b` in my input but `0x55` in the encrypted flag. This must be why the check is failing.

But the problem is that we have correctly used `0x55` in our encrypted_message. Let's try replacing `0x55` with `0x5b` and see what happens.

## Final exploit

This is the new exploit script with the modified encrypted_message.

```python
sigma_words = [0x61707865, 0x3320646E, 0x79622D32, 0x6B206574]

# Function to derive key_bytes from sigma_words
def derive_key_bytes():
    key_bytes = []
    for i in range(4): 
        w = sigma_words[i]
        key_bytes.append(w & 0xFF)          # Low byte
        key_bytes.append((w >> 8) & 0xFF)   # 2nd byte
        key_bytes.append((w >> 16) & 0xFF)  # 3rd byte
        key_bytes.append((w >> 24) & 0xFF)  # High byte
    return key_bytes

# Decryption function
def sigma_decrypt(encrypted_message, key_bytes):
    decrypted_message = bytearray(len(encrypted_message))
    for i in range(len(encrypted_message)):
        decrypted_message[i] = encrypted_message[i] ^ (i + key_bytes[i & 0xF])
    return decrypted_message


# flag_enc instead of message
encrypted_message = bytearray([0x03,0x15,0x13,0x03,0x11,0x5b,0x1f,0x43,0x63,0x61,0x59,0xef,0xbc,0x10,0x1f,0x43,0x54,0xa8])

key_bytes = derive_key_bytes()

decrypted_message = sigma_decrypt(encrypted_message, key_bytes)

print("Decrypted message:", decrypted_message.decode())
```
Getting the decrypted message and checking it against the binary again.

```bash
$ python ez-flag-exploit.py
Decrypted message: flagc29yYW5k<b19!!
$ ./chall
Enter flag: SECCON{flagc29yYW5k<b19!!}
correct flag!
$ 
```

And we have it! The correct flag is `SECCON{flagc29yYW5k<b19!!}` and we have solved the `ez_flag_checker` challenge by SECCON 14 CTF Quals. Thank you for reading and I will see you in the next writeup!