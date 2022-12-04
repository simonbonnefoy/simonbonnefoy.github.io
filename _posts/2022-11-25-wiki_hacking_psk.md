---
layout: post
title:  "Cracking WPA2 wifi"
date:  2021-05-23 12:00:00 -500
tags: [ brute force, penetration testing, cryptography, hacking, penetration testing, wifi]
categories: [ brute force, penetration testing, cryptography, hacking, penetration testing, wifi]
---

# How is WPA2 cracked

Since a while, I have been wanting to get into some wifi hacking. It looks like this is one of the skill that everybody would like to get. Just be able to get any wifi, anywhere you are. Well, maybe that was true some years ago, I am not sure this is still so easy today. Still, I wanted to get into it and try by myself. Mainly, I wanted to understand how it was possible to crack some wifi. By asking your favorite search engine about cracking wifi, you might run into a lot of tutorials showing you which commands to copy and past in kali linux in order to start hoping you might crack some wifi around you. But, beside this empowering feeling you get when learning to hack wifi, what is more interesting is how is it possible to crack a wifi network, and where is the security flaw. What is the magics that the aircrack-ng suite is relying on? Let's take a look at it.

## A word on Pre-Shared Key (PSK)

In this post, I will stick to WPA2-PSK wifi, since it is the most common that is found nowadays.
PSK stands for pre-shared key. This means that the key is previously known from the station (STA), i.e., your computer, smartphone, etc. and the access point (AP), i.e, your router. When you try to connect to a WPA2-PSK network, you will be asked to enter passphrase to validate that you are a genuine user. That passphrase is note directly transmitted to the AP to check if you are a legitimate user. It is used to compute a pre-shared key (PSK). The PSK is computed using a Password-Based Key Derivation Function 2 (PBKDF2). That functions takes as input a pseudo-random function (HMAC-SHA1 for WPA2), the passphrase of the network, the name of the network (SSID), its lenght,, the number of iterations (4096) and the size of the PSK we want to generate (bytes). So, imagine that you have a wireless network with the name MyNetwork and the passphrase my_secret, the PSK will look like:
`PSK = PBKDF2(HMAC-SHA1, 'MyNetwork', 'my_secret', lenght('my_secret'),4096,32)`

## The 4-way handshake
When connecting to an AP, the STA will first send a probe request to the AP, to which the AP will respond sending information about the wireless network, i.e., speed, encryption type, etc. After this point, since the STA knows the characteristics of the AP, they can both agree on the cipher suite to be used. After agreeing to use WPA2-PSK, a process called the 4-way handshake will start. The goal of the 4-way handshake twofold: first, the AP checks whether the STA has the good passphrase, and second, the exchange of the keys used for encryption. The 4-way handshake goes as follow:
* The AP generates a random number used only once (NONCE) and sends it to the STA. Let's call this number ANONCE (AP NONCE).
* The STA creates a SNONCE (STA NONCE) and will compute the pairwise transient key (PTK) using as input the PWK, ANONCE, SNONCE, STA MAC address and AP MAC address. Since the STA has been communicating with the AP, it already knows its MAC address. After generating the PTK, the STA will send the SNONCE and a message integrity check (MIC). The MIC is computed using the PTK freshly computed. The PTK will later be used for data encryption between the STA and the AP.
* The AP uses the SNONCE that it receives from the STA to compute the PTK, using the same parameters as the STA. The AP then checks the MIC with the PTK is has computed. If the MIC check is successful, it means they both have the same PTK, hence, they both have the same PWK, since it was used to derive the PTK. Now, the AP can send the group transient key (GTK) to the STA with a MIC. The GTK is used for broad/multi-cast communication over the network.
* The STA sends an acknowledgement to the AP after receiving the GTK.
![4-way handshake description](../_data/images/wpa2.png)

After these steps, the STA and AP can communicate sending encrypted packets.

## Cracking WPA2-PSK
So, now that we have seen the process on which relies the WPA2-PSK protection, let's see how it can be cracked. The goal, is to reproduce the MIC from the STA. If we are able to do it, it means we have the good keys (PTK, PWK and passphrase). Since the PTK is not verified before the 4-way handshake, some data exchange will have to appear in clear text (not encrypted). Thus, it is easy for anyone sniffing the traffic to get the 4-way handshake of an STA that is trying to connect to a wireless network. While sniffing the packets during the process, an attacker can easily get:
* SSID
* STA MAC address
* AP MAC address
* ANONCE
* SNONCE
* First MIC coming from the STA
With all this knowledge, the only unknown to reproduce the MIC that the AP has to check is the passphrase. The only solution here is a brute-force dictionary attack.We have to reproduce the 4-way handshake process for each entry in the dictionary, and check the MIC value. That's how for example the aircrack-ng suite is working. This process can take quite some time and requires strong computing resources. However, the process can be made faster using so-called rainbow tables.

## Rainbow tables

One of the bottleneck in the WPA2-PSK brute-force attack, is the calculation of the PMK, due to the call of the PBKDF2 function. The rest of the attack is much faster to execute. However, the results of the PBKDF2 function will only depend on the passphrase and the SSID. Hence, it is possible to compute and store in a database the PMK for a given SSID. The tables used to store the PMK for a given SSID are called rainbow tables. One can compute a rainbow table for a given SSID on a powerful server, and use it to crack WPA2-PSK on a less powerful machine. Also, if two wireless networks of interest have the same SSID, the same rainbow table can be used on both networks.

## How to mitigate attack against WAP2-PSK
As we saw, the bottleneck in the brute-force attack of WPA2-PSK is the calculation of the PMK. Thus, making that process hard should help mitigating an attack. For that two things can be done. First, use a strong passphrase. And yes, you must use a PHRASE. Something long and complicated that will require time to be cracked. Pay attention, long an complex does not mean something impossible to type or to remember, but rather something hard to guess. A phrase such as "I like when the sun is shinning through my windows" is easy to remember and to enter, and will take a crazy huge amount of time to break. Take a look here to try and see what kind of passphrase might require some time to be cracked. Second point, we saw that rainbow tables need the SSID to be computed. If your AP is the one that was given to you by your internet service provider (ISP), it will most likely have a standard name, and probably someone will have pre-computed some rainbow tables for the AP of this brand. The best thing, is to change also the SSID of you access point. This will help mitigate brute force attack using rainbow table, since a set of rainbow table will likely have to be computed only for your AP if you find an original SSID.
