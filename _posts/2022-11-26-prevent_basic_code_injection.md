---
layout: post
title:  "Prevent basic code injections"
date:  2021-08-23 12:00:00 -500
tags: [ brute force, penetration testing, cryptography, hacking, penetration testing, wifi]
categories: [ brute force, penetration testing, cryptography, hacking, penetration testing, wifi]
---

# How to prevent basic code injections

Injections are code instructions that are executed somewhere not expected. A famous case is SQL injection, where an user can inject instructions that will be interpreted by a database management system, whereas this user is not meant to directly interact with the database nor execute code on the database server. Code injection, could, theoretically, take place anywhere during a code execution where the user is asked for input, if the code is not properly sanitized. Code injection can happen with different programing languages (C, C++, python, php, etc).

## How to spot vulnerabilities
By taking a quick look at the code that is run, one can try to spot some place for code injection.
Take for example the python code below:
```python
import os

file = input('Select a file to print ')
os.system(('cat %s' %file))
```
Let's put this code in a file, let's say code_injection.py.
This code asks the user for a file to print and prints it in the terminal. The `os.system()` instruction makes a system call to execute a shell command. Let's print the test.txt file that contains the following message: "Hi, your python script is working!"

```bash
root@localhost:~$ code_injection.py
Select a file to print: test.txt
Hi, your python script is working!
```

The python script prints out the content of the test.txt file as expected. What the script does inside, is to call the `system()` command with the `os.system()` instruction . Basically, the `os.system()` function will execute the shell instructions given in argument. Shell instructions are given in the bash language. In bash, like in many other programing languages, you can chain commands on one line using separating characters. In bash, the separating character is semi-column ';'. You can for example couple several commands like this:
```bash
root@localhost:~$ whoami; pwd
root
/root
```

This can also be done in the os.system() call in our python script.
```bash
root@localhost:~$ python code_injection.py
Select a file to print test.txt; whoami; pwd
Hi, your python script is working!
root
/root
```
You see, in this case, the script reads the file and also executes the command we put after the separating characters. In this case, we just asked the script to tell us who we are and in which folder we are, but we could ask for anything the user is able to do. This kind of code injection can be found, for example, on a website where there is a field in which the user can directly type into. For example a search field on an online-shop, where the user can type the product he is looking for. The web server then will have to send a query to the database to retrieve the product's price, quantity, etc. If the user knows which database management system is running in the background, he might be able to inject some commands that can be interpreted directly by the database and so, interact with the database as he wishes.

## How to prevent code injection
Basic code injection can be avoided using simple exclusion rules. If your code involves some calls to the `os.system()` method in python, this mean that it deals with some bash instructions. As we saw, the command separator in bash is the semi-column character ';'. Then a quick check can tell us if such a character is present in what the user has typed. If a non desired character is given, then you can throw an error saying that an invalid character was entered. You can do the same for the commenting characters. Another solution would be to establish a regexp that would discard any argument that does not look like what the code is expecting.
