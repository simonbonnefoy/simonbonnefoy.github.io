---
layout: post
title:  "HTB BabyEncryption write-up"
date:  2022-12-03 12:00:00 -500
tags: [cryptography, python]
categories: [cryptography, python]
---

# HTB - BabyEncryption write-up
The [BabyEncryption](https://app.hackthebox.com/challenges/babyencryption) challenge is one of the entry level crypto challenges at [HackTheBox](https://app.hackthebox.com).

You need to download the file and unzip it and you get a `chall.py` and `msg.enc` files.
By taking a look at both file, it seems that the `msc.enc` file is a ciphered message that was encrypted using the `chall.py` code.
The `msc.enc` file contains the following ciphered message
```
6e0a9372ec49a3f6930ed8723f9df6f6720ed8d89dc4937222ec7214d89d1e0e352ce0aa6ec82bf622227bb70e7fb7352249b7d893c493d8539dec8fb7935d490e7f9d22ec89b7a322ec8fd80e7f8921
```
The `chall.py` file contains the following code:
```python
import string
from secret import MSG

def encryption(msg):
    ct = []
    for char in msg:
        ct.append((123 * char + 18) % 256)
    return bytes(ct)

ct = encryption(MSG)
f = open('./msg.enc','w')
f.write(ct.hex())
f.close()
```
Here we can see how was ciphered the original message.
There are two steps in this process:
* The message is ciphered through the `encryption` function
* The ciphered message is converted to hexadecimal before being written to file.

We need to revert these two steps to get to the original message.

## Decryption
The first step is to convert the string back from hexadecimal.
This is done using the `bytes.fromhex()` function :
```python
f = open('./msg.enc','r')
cipher =f.read()
cipher_fromhex =bytes.fromhex(cipher)
print(cipher_fromhex)
```
This piece of code returns the following bytes
```python
b'n\n\x93r\xecI\xa3\xf6\x93\x0e\xd8r?\x9d\xf6\xf6r\x0e\xd8\xd8\x9d\xc4\x93r"\xecr\x14\xd8\x9d\x1e\x0e5,\xe0\xaan\xc8+\xf6""{\xb7\x0e\x7f\xb75"I\xb7\xd8\x93\xc4\x93\xd8S\x9d\xec\x8f\xb7\x93]I\x0e\x7f\x9d"\xec\x89\xb7\xa3"\xec\x8f\xd8\x0e\x7f\x89!'
```
You can iterate over each byte and the base 10 value for each element:
```python
for i in cipher_fromhex:
    print(i)
```
This returns
```
110
10
147
114
236
73
...
```
Now we need to revert the `((123 * char + 18) % 256)` equation, which is at the heart
of the cryptographic process here.

### Modular arithmetic
The equation `((123 * char + 18) % 256)` is a modular arithmetic operation.
It can be written as
```
123 * c + 18 ≡ X (mod 256)
```
Where `c` is the clear text and `X` is the ciphered text.
This operation is known as [affine cipher](https://en.wikipedia.org/wiki/Affine_cipher).
It can easily be reverted.
```
123 * c + 18 ≡ X (mod 256)
123 * c ≡  X - 18 (mod 256)
c ≡  123^(-1)* (X - 18) (mod 256)
```
We can isolate the clear text element and invert the equation.
Here the term `123^(-1)` represents the modular inverse of `123`.
This means that `123 * 123^(-1) ≡ 1 (mod 256)`

Let's compute the inverse:
```python
for i in range(0,256):
    if (i * 123) % 256 == 1:
        print(i)
```
This code will test all the integers from 0 to 256 and check if they are the inverse value of 123.
Note that is an acceptable method for small number. With bigger numbers we'd need to use the [Extended Euclidian Algorith](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm).
This gives us `123^(-1) = 179`. We now have the complete decryption equation:
```
 c ≡  179 * (X - 18) (mod 256)
```
### Decryption function
We can write the decryption function as follows
```python
def decryption(ciph):
  msg = ""
  for b in ciph:
    char = (b -18)*179 % 256
    msg += chr(char)
  return msg
```

The full code would look like this
```python
def decryption(ciph):
  msg = ""
  for b in ciph:
    char = (b -18)*179 % 256
    msg += chr(char)
  return msg

if __name__ == '__main__':
  f = open('./msg.enc','r')
  cipher =f.read()
  cipher_fromhex = bytes.fromhex(cipher)
  msg = decryption(cipher_fromhex)
  print(msg)
```
This returns the string
```text
Th3 nucl34r w1ll 4rr1v3 0n fr1d4y.
HTB{l00k_47_y0u_r3v3rs1ng_3qu4710n5_c0ngr475}
```
And we have our flag.
