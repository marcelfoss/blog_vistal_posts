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

More Text..

``` sh
sqlite3 $filename".db" 'CREATE TABLE performance (algorithm TEXT, file_size TEXT, time REAL);'
sqlite3 $filename".db" ".mode csv" ".import --skip 1 time_checksum.csv performance" ".exit"
```
