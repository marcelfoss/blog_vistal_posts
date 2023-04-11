+++
author = "Marcel Peters"
title = "Problems with data degradation and their consequences for financial data."
date = "2023-04-11"
description = "problems and consequences with data degradation / bit rot."
tags = [
    "data security",
    "scripts",
    "checksum",
    "linux",
    "financial data",
    "excel"
]
+++

The problem is that a random bit flip can change data entirely. But the type of data matters for us. With financial data, for instance, it can result in dramatically inaccurate numbers.

<!--more-->
---

# Data degradation


## Table Of Content

1. The Problem
2. The Occurrence
3. Performance Comparison
4. Conclusion
5. Reference Sources

## The Problem

We are dealing with data everyday. No matter what kind of data you are processing right now. Even this file is made of a data, basically streamed in binary form to construct (render) this website. Data is represented internally in binary, most of us already know this, yet if we look at the representation of a primitive data type, for instance an integer, we quickly realize that a single change in the stored value (switching a 0 to a 1) can lead to change the number dramatically.  
   
Lets have a look at an example:  
In many representations of integer values the data is stored in 32bit values.[1] 32bit values means we have 32 binary values in a row. Looking at the last deal one of our sales rep made:  
He closed the deal with an value of USD __8,000__. This number gets represent in binary of the value:  
`00000000000000000001111101000000`.  
If we 'flip' a random bit somewhere in the middle our value is represented as: `00000001000000000001111101000000`.  
If we transform it in a more human-friendly form (decimal system): USD __16,785,216__.  
  
  
  
   <img src="https://media.vistal.io/images/blog/excel.png" alt="Figure 2: wrong value in a spreadsheet." width="500" />  
     
 _Figure 1: wrong value in a spreadsheet._

Not too bad. Someone might be happy. See, the problem is that a random bit flip can change data entirely. But the type of data matters for us. For some data types it is not really a problem. If we look at a JPEG shot by our smart phone, with a 12MP camera, we have 12,000,000 pixel. If we have a bit flip here we arguable wouldn't really notice the difference, do we? But financial data, for instance, is different.

## The Occurrence
But how does data degradation occur?  
There are many reasons for that. In general, nearly all media is affected by so called 'bit rot'.

>With conventional hard drives, for example, wear and tear, extreme temperatures or the effects of electromagnetic radiation can cause individual bits to tilt. On flash-based storage media such as solid state drives (SSD), the loss of electrons in the storage cells is a possible cause of data rot. Optical data carriers such as CDs, DVDs or Blu-rays suffer from gradual data corruption, for example due to the ageing of the data carrier material used or from external mechanical damage impacts. [2]

Modern server file systems calculated or make use of other techniques to check if the data integrity is still given. For the Linux operating doing this we have Btrfs; Free-BSD comes with its famous OpenZFS implementation and ReFS when we are talking Windows. Those file systems will digest the bit-stream, write the data to the media, and write the result of the digest as well on the media. If we are then trying to access the file, in the background, the operating system will do the same calculation and compare the results.[3],[4]  
If we have a look at the documentation for Btrfs for instance we see that there are several algorithms which we can use to compute those checksums. But why should we even bother with which to use?  
Next to data collision, if we have two different inputs resulting in the same checksum,[5] we are likely to have the same problem we wanted to mitigated in the first place. Performance on the other hand can also play a significant role as an higher impact on the CPU might lead to a less responsive system. But how likely is a collision?

> MD5: The fastest and shortest generated hash (16 bytes). The probability of just two hashes accidentally colliding is approximately: 1.47*10^-29.

> SHA1: Is generally 20% slower than md5, the generated hash is a bit longer than MD5 (20 bytes). The probability of just two hashes accidentally colliding is approximately: 1.47*10^-29

> SHA256: The slowest, usually 60% slower than md5, and the longest generated hash (32 bytes). The probability of just two hashes accidentally colliding is approximately: 4.3*10^-60.[6]

Well, this does not sound too bad at all at first glance. But as we remember, Btrfs, which is our best bet in the Linux space, uses CRC32 by default. And quite frankly, most server which follow the default setup routine are using a system which is not doing checksumming at all - such as XFS on Redhat Linux Enterprise. So lets see how the best case scenario out of the box for our file server compares to a tuned configuration. If we compare it, its more likely to have a collision with CRC32 than having a 4 of a kind in poker[7] - which in fact happens often enough that this might be a concern of your precious financial reports, isn't it? _(Annotation from the Author: MD5 is, because of it's probability for collision, already considered as broken for security purposes.)_

## Performance Comparison
So as we aren't willing to implement a solution which is not satisfying enough (otherwise we could again stick to not doing anything at all) let us compare  how the performance penalty actually is on our system. Please note that this is a comparison on one machine and can have a different result on other machines due to instruction sets and other factors. As for specific problems, especially CISC processors, are having edges with specific instructions. If you are interested in how this compares on your server or workstation I uploaded the source code of my script so you can download it to test it on your own.[8]

For this we will generate 3 files with random content:

- 100 KiB - representing smaller files such as JPEGs
- 100 MiB - representing media files such as small videos and music files
- 1 GiB   - representing large files such as bigger videos, ISO files, ..

As our file system will also perform checksum on larger files those are also part of our consideration.  
The algorithm is designed to do 5 rounds of checksumming for each file to reduce the margin of error.  
After that we will aggregate the data and have a look on the result:

| algorithm | data size | time in seconds |
|-----------|---------:|----------------:|
| sha1      |     100M |           0.062 |
| sha256    |     100M |            0.07 |
| b2        |     100M |           0.113 |
| md5       |     100M |          0.1256 |
| sha512    |     100M |          0.1386 |
| sha1      |       1G |          0.5944 |
| sha256    |       1G |          0.6744 |
| b2        |       1G |          1.1054 |
| md5       |       1G |          1.2392 |
| sha512    |       1G |          1.3674 |
| b2        |       1K |           0.001 |
| md5       |       1K |           0.001 |
| sha1      |       1K |           0.001 |
| sha256    |       1K |           0.001 |
| sha512    |       1K |           0.001 |

_Figure 2: Performance results_

As we see, on my machine, SHA256 has nearly the same performance as SHA1, so there is not really a reason to go _(Annotation by the Author: also broadly considered broken)_ with SHA1. Even MD5 is significant slower than SHA256 on my machine. If we would just have used the algorithm with the shortest checksum we would have had a slower algorithm used - although we might have expected to be the quickest. SHA512 and BLAKE2B have a significant higher performance hit for larger files as well. In this scenario the balance (performance/result ratio) is in favour of SHA256 on my machine. Quick reminder, Gorka Remirez[6] found SHA1 to be 20% slower than MD5 which in fact does not apply to our findings here.


## Conclusion
When dealing with data, I need to rely on their integrity; I should make some consideration on how I want to archive this. It doesn't matter if these are my Excel files I use or other formats. Maybe I just perform my checks on archived data or I do automatised checks in form of scripts or on the file system level. Whatever you decide to do small errors will most likely occur at some point.  
If you want to learn more or need help with this or any other (data)-related topics, feel free to send me a message on [LinkedIn](https://www.linkedin.com/in/marcel-e-peters/).

## Reference Sources


* [1] [IBM. (2022). INTEGER data type. Retrieved on 11. April 2023](https://www.ibm.com/docs/en/informix-servers/14.10?topic=types-integer-data-type)
* [2] [Stefan Luber. (09.03.2023). Was ist Bit Rotting / Data Rot / Silent Corruption? (German original source; translated into English) Retrieved on 11. April 2023](https://www.storage-insider.de/was-ist-bit-rotting-data-rot-silent-corruption-a-56c425eb3c0302b2455133e8211ddaef/)
* [3] [Btrfs man(5). CHECKSUM ALGORITHMS. Retrieved on 11. April 2023](https://btrfs.readthedocs.io/en/latest/btrfs-man5.html#checksum-algorithms)
* [4] [FreeBSD Docs. Retrieved on 11. April 2023](https://docs.freebsd.org/en/books/handbook/zfs/#zfs-term-checksum)
* [5] [BSI. What is checksum verification? Retrieved on 11. April 2023](https://www.bsi.bund.de/dok/6599444) 
* [6] [Gorka Ramirez. (2015) MD5: The broken algorithm. Retrieved on 11. April 2023](https://www.avira.com/en/blog/md5-the-broken-algorithm)
* [7] [Preshing on Programming. (2011). Hash Collision Probabilities. Retrieved on 11. April 2023](https://preshing.com/20110504/hash-collision-probabilities/)
* [8] [Marcel Peters. (11.04.2023). Source Code Github Gist. Retrieved on 11. April 2023](https://gist.github.com/marcelfoss/2ab51f3c3917296ebbbbce751d7b1cd5)
