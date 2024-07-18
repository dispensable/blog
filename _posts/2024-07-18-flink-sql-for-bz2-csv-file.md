---
layout: post
title: Flink sql csv connector need \r\n newline terminator when csv is compressed with bz2 format
tags:
- flink
---

Flink sql can query csv file with \r newline terminator. but if you compressed csv file it just can't work (you can query some records but not all which is confused).
so i asked a question in the flink slack channel.

> hi, all. Is it possible to process compressed(bz2) csv file with flink sql filesystem connector ?
> I can select some records but only a few of the dataset) even with raw format.
> Is the bz2inputstream read the data as 'line' or just block size of the bz2 file ?

ok, i guess i found the reason. flink csv connector will got data stream from bz2inputstream which requires \r\n as the newline terminator. 
but the plain csv file can use \n newline terminator also. just rewrite the csv file with \r\n and it works
