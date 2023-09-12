---
title: "Cyberheroines CTF 2023"
description: "A cybersecurity competition that pays homage to women’s unique contribution to computer science and cybersecurity."
tags: [ctf]
date: 2023-09-11
math: false
---

The Cyber heroines CTF competition is a cybersecurity competition that pays homage to women’s unique contribution to computer science and cybersecurity. It is built by the Florida Tech WICYS chapter in partnership with the L3 Harris Institute for Assured Information and FITSEC cybersecurity team.

The categories are: pwn, re, crypto, web, forensics

The challenges are on a beginner level, I will focus on the web challenges only. Each challenge is named after a highly influential female computer science researcher, who will be introduced alongside the challenge solution.

## Grace Hopper

![Grace Hopper](ctf/cyberheroines/images/hopper-profile.png)

Grace Hopper, born in 1906, was a pioneering computer scientist and U.S. Navy Rear Admiral. Her most significant achievement was the development of the first compiler, a program that translated human-readable code into machine code. This innovation revolutionized programming, making it more accessible and efficient. Hopper's work laid the foundation for modern computer languages and played a crucial role in the early days of computing. She was also among the team of scientists who found the first computer bug, literally. A moth was trapped in their computer at Harvard.

![First computer bug](ctf/cyberheroines/images/bug.jpg)

In the CTF challenge, we can input commands into an input field and somehow have to retract the flag. But soon to be noticed, a lot of linux commands are blogged by the system, including ls, cat, head and many more. Still, this challenges showcases that it is relatively easy to circumvent blacklisting of "security critical"commands.
A quick google search helped me to find that I can list the files and read them using

```bash
echo *

while read line;
do echo $line;
done <cyberheroines.sh
```

This reveals the flag.

## Susan Landau

![Susan Landau](ctf/cyberheroines/images/landau-profile.png)

Susan Landau is a distinguished figure in the field of cybersecurity and digital privacy. Her most notable achievement is her influential work in advocating for strong encryption and privacy protections in the digital age. She has been a vocal proponent of safeguarding individuals' rights to secure communication and has made significant contributions to the development of encryption policies and technologies. She has testified in front of the US congress and written for the Washington Post among others.

![Hints on the webpage](ctf/cyberheroines/images/landau-1.png)

While navigating through the webpage associated with this challenge, we get some hints: Something about deciphering a secret code and using the "cyberheroine" username.

After analyzing the whole application by looking at the source code and utilizing Chrome Developer Tools, we find a CSRF token. With Crackstation, we obtain it's value "Hack this" and learn that it is a MD5 hash.

![Cracking the hash](ctf/cyberheroines/images/landau-3.png)

If we now go into Burp Repeater and make another request, but substituting that CSRF token with the hash of "cyberheroine", we obtain the flag as the response.

## Radia Perlman

![Radia Perlman](ctf/cyberheroines/images/perlman-profile.jpg)

Radia Perlman, or the "mother of the internet", is a renowned computer scientist celebrated for her groundbreaking work in the field of network protocols. Her most significant achievement is undoubtedly the creation of the [Spanning Tree Protocol (STP)](https://www.techtarget.com/searchnetworking/definition/spanning-tree-protocol), a fundamental protocol that ensures the stability and redundancy of computer networks.

In this challenge, we get presented a webpage called "My DNS App" and we get told that we can query the DNS information of any domain via the dns query parameter like so:

```bash
curl https://cyberheroines-web-srv3.chals.io/dns?ip=cyberheroines.ctfd.io
```

![Radia Perlman](ctf/cyberheroines/images/perlman.png)

The output looks a lot like it just runs an OS command in the background, so it is natural to try to inject additional OS commands to read out the flag, e.g.

```bash
curl https://cyberheroines-web-srv3.chals.io/dns?ip=cyberheroines.ctfd.io;ls
```

This shows us in the output that there is indeed a flag.txt.

The cat command is blocked. However we can circumvent it using the following trick to get the flag:

```bash
# grep "" flag.txt
curl https://cyberheroines-web-srv3.chals.io/dns?ip=cyberheroines.ctfd.io;grep%20%22%22%20flag.txt
```

## Shafrira Goldwasser

![Shafrira Goldwasser](ctf/cyberheroines/images/goldwasser-profile.jpg)

Shafrira Goldwasser is a computer scientist with groundbreaking achievements in the field of cryptography and theoretical computer science. One of her most significant contributions is in the development of cryptographic protocols that ensure secure communication and protect sensitive information in an increasingly digital world.

Goldwasser's pioneering work spans various aspects of cryptography, including the development of [zero-knowledge proofs](https://www.youtube.com/watch?v=fOGdb1CTu5c), which are cryptographic methods that allow one party to prove to another that they possess certain information or knowledge without revealing the actual content of that knowledge.

For this challenge, we get a webpage where we can query biography's of a given list of of female computer scientists.

![Source code](ctf/cyberheroines/images/goldwasser-2.png)

But in contrast to the previous challenge, we also get to see the source code.

At first, it looks like a naive SQL injection, but actually it is another Command Injection, where we just have to escape the query string using _'" &&_ and after that we can run arbitrary linux commands again.

![In Burp](ctf/cyberheroines/images/goldwasser-3.png)

Sending the request for the biography to Burp Repeater, we can substitute the _heroine-name_ parameter with the following payload and get the flag:

```
ada'" && cat /flag.txt #
```

## Frances Allen

![Frances Allen](ctf/cyberheroines/images/allen-profile.jpg)

Frances Allen was a trailblazing computer scientist known for her remarkable achievements in the field of compiler technology and parallel computing. Her most significant accomplishment was her pioneering work in optimizing compilers, which are essential software tools that translate high-level programming languages into machine code. Allen's innovations greatly improved the efficiency and performance of computer programs, making them run faster and consume fewer resources.

![Frances Allen](ctf/cyberheroines/images/allen.png)

On the webpage of the challenge, we can make a post to nominate our personal cyber heroine.

Trying to enter something like _{{2 + 2}}_ and getting _4_ back confirms that we have a template injection vulnerability here. Given that all the previous challenge was written in Python Flask, we can assume that it is probably the Jinja templating language that is used.

We can find a suitable payload to read out the flag from [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md#jinja2---basic-injection):

```
{{ get_flashed_messages.__globals__.__builtins__.open("/flag.txt").read() }}
```

This concludes the challenges in the web category!
