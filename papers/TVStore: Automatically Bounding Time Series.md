---
link: https://www.usenix.org/system/files/fast22-an.pdf
---

We consider the problem of automatically bounding the storage of a time series store by compression. To enable this, our key insight is that time series data can be compressed losslessly or lossily according to its importance, which is in turn related to its age, as users commonly accept information loss on less important old data.

The importance of time series data changes along with time, as reflected by applications’ favoring recent data over old data, or favoring some events at certain moments over others.

**For important data, we compress them losslessly or with a low ratio by lossy compression. For unimportant data, we compress them by a high compression ratio.**

Later rounds of compression must execute on differently compressed data.

**If we compress already compressed data the compression ratio grows exponentially.**

**Decompression before compression is highly inefficient.**

If lossy compression is used, the decompressed data is imprecise. Rounds of decompression and compression can lead to a high deviation from the original data.

To avoid the above two problems, TVC requires the
compressor to have the following three properties: 

1) Compression on previously compressed data does not require
decompression 
2) Decompression on data compressed multiple times works the same way as on data compressed once
3) The error bounds must be easily computed for the rounds of compression

According to the related work, a line segment built from two line segments is the same as the line segment built from the original time series data, if line segments are properly constructed. Decompression on data at any round only needs to compute the linear function for a given time. Moreover, the mean bias error (MBE) of PLA can be computed easily even after rounds of compression.
