+++
author = "Marcel Peters"
title = "Data Integrity"
date = "2023-04-11"
description = "A brief example description."
tags = [
    "data security",
    "scripts",
]
+++

Intro text

<!--more-->
---


- problem mit bitrot und die folgen fuer financial data
# Data degradation

We are dealing with data everyday. No matter what kind of data you are processing now. Even this file is made of a lot of data, basically streamed in a binary form to construct this webpage. Data is represented internally in binary, most of us already know this, yet if we look at the representation of a primitive data type, for instance an integer, we quickly realise that a single change in the data (switching a 0 to a 1) can lead to changing the number dramatically.[1] Lets have a look at an example:
In many representations of integer values the data is stored in 32bit values. 32bit values means we have 32 binary values in a row. Let's look at the last deal one of our sales rep made. He closed the deal with an value of USD 8,000. This number gets represent in binary of the value `00000000000000000001111101000000`. If we 'flip' a random bit somewhere in the middle our value is represented as '00000001000000000001111101000000'. Lets look what this is if we transform it in a more humanfriendly form (decimal system): 16,785,216. Not too bad. Someone might be happy. See, the problem is that a random bit flip can chance data enterly. No matter what data. For some data types it is not really a problem. If we look at a JPEG shot by our smartphone with a 12MP camera we habe 12,000,000 pixel. If we have a bit flip here we arguable wouldnt really notice the difference - do we? But financial data, for instance, is different.
But how does data degredation occur?
There are many reasons for that. In general nearly all media is affected by so called 'bit rot'.
*With conventional hard drives, for example, wear and tear, extreme temperatures or the effects of electromagnetic radiation can cause individual bits to tilt. On flash-based storage media such as solid state drives (SSD), the loss of electrons in the storage cells is a possible cause of data rot. Optical data carriers such as CDs, DVDs or Blu-rays suffer from gradual data corruption, for example due to the aging of the data carrier material used or from external mechanical damage impacts.*
[2]

Modern Server Filesystems calculations or other technics to check if the data integrity is still given. For the Linux Filesystem we do have Btrfs; FreeBSD comes with its famous OpenZFS implementation and ReFS when we are talking Windows. Those filesystems will digest the bitstream, write the data to the media, and write the result as well on the media. If we are then trying to access the file, in the background, the operating system will do the same calculation and compare the results.[3],[4]
If we have a look at the documentation we see that there are several algorithms which we can use to compute those checksums. But why should we even bother with which to use?  
Next to data collision, if we have two different inputs resulting in the same checksum,[5] we are more likely to have the same problem we wanted to medigate in the first place. Performance on server can also play a significant role as such higher impact on the CPU and hence less responsive systems. Lets have a look on how likely a collision is by algorithm[6].
*MD5: The fastest and shortest generated hash (16 bytes). The probability of just two hashes accidentally colliding is approximately: 1.47*10-29.

SHA1: Is generally 20% slower than md5, the generated hash is a bit longer than MD5 (20 bytes). The probability of just two hashes accidentally colliding is approximately: 1*10-45

SHA256: The slowest, usually 60% slower than md5, and the longest generated hash (32 bytes). The probability of just two hashes accidentally colliding is approximately: 4.3*10-60.*

Ok this does not sound to bad at first. But as we remember, Btrfs, which is often used in the linux space uses CRC32 as a default. And quite frankly most server which follow the default routine are using a system which is not doing checksumming at all - such as XFS on RHEL. So lets see how the best case scenario out of the box for our fileserver compares. If we compare it, its more likely to have a collision with crc32 than having a 4 of a kind in poker - which in fact happens often enough that this might be a concern of your precisious financial reports, isn't it? (Annotation of the Author: MD5 as because of the probabilty already considered broken for security reasons).

So as we aren't willing to implement a solution which is not satisfying (otherwise we could again stick to not doing anything at all) let us compare actually how the performance penality is on our system. Please note that this is a comparison on one machine and can have a different result on other machines due to instruction sets and other factors as for specific problems, especially CISC processors, are having edges with specific instructions. If you are interested in how this compares on your server or workstation I uploaded the source code of my script so you can download it to test it on your own. [8]

For this I will generate 3 files with random content:
100K - representing smaller files such as JPEGs
100M - representing media files such small videos and music files
1G - representing large files such as bigger videos, iso files, ..

As our filesystem will also perform checksum on larger files those are also part of our consideration.
The algorithm is designed to do 5 rounds of checksumming for each file to reduce the margin of error.
After that we will aggregate the data and have a look on the result:

algorithm| datasize| time in seconds
sha1| 100M|0.062
sha256| 100M|0.07
b2| 100M|0.113
md5| 100M|0.1256
sha512| 100M|0.1386
sha1| 1G|0.5944
sha256| 1G|0.6744
b2| 1G|1.1054
md5| 1G|1.2392
sha512| 1G|1.3674
b2| 1K|0.001
md5| 1K|0.001
sha1| 1K|0.001
sha256| 1K|0.001
sha512| 1K|0.001

As we see, on my machine, SHA256 has nearly the same performance as SHA1, so there is not really a reason to go (also considered broken) with SHA1. Even MD5 is significant slower that SHA256 on machine. If we would just have used the alogirtihm with the shortest checksum we would have had a higher impact - although we might have expected to be the quickest. SHA512 and blake2b have a significant higher performance hit for larger files as well. In this scenario the balance (performance/result ratio) is in favour of SHA256 on my machine.


As a conclusion:
When dealing with data I need to rely on their integrity I should make some consideration on how I want to archive this. It doesn't matter if these are my Excel files I use or other formats. Maybe I just perform my checks on archived data or I do automatised checks in form of scripts or on the data system level. Whatever you decide to do small errors will most likely occur at some point. If you have questions how it can be archived in your organisation feel free to write me a message on LinkedIn.




TL;DR Dealing with financial data? do NOT use md5 or crc32 for checksumming in 2023

[-] [Binary Representation of Integers - Nischal Babu Bohara - 2020](https://medium.com/@nischalbohara77/binary-representation-of-integers-bc7f52c21202) - Quelle schlecht
[2] [Schleichende Datenkorruption - Dipl.-Ing. (FH) Stefan Luber 2023](https://www.storage-insider.de/was-ist-bit-rotting-data-rot-silent-corruption-a-56c425eb3c0302b2455133e8211ddaef/)
[3] [Btrfs manpage-5-](https://btrfs.readthedocs.io/en/latest/btrfs-man5.html#checksum-algorithms)
[4] [https://docs.freebsd.org/en/books/handbook/zfs/#zfs-term-checksum](FreeBSD Docs)
[5] [What is checksum verification? - BSI](https://www.bsi.bund.de/dok/6599444) 
[6] [MD5: The broken algorithm - Gorka Ramirez - 2015](https://www.avira.com/en/blog/md5-the-broken-algorithm)
[7] [Hash Collision Probabilities - Preshing on Programming - 2011](https://preshing.com/20110504/hash-collision-probabilities/)
[8] [Source Code]
