---
layout: post
title:  "LZ4 streamed processing"
date:   2017-10-04 22:10:39 +0200
categories: blog
excerpt_separator: <!-- more -->

---

Processing terabyte large LZ4 files in a streaming fashion with resumption and random access.

<!-- more -->

## Intro

LZ4 is lossless compression algorithm, providing compression speed at 400 MB/s per core, scalable with multi-cores CPU. It features an extremely fast decoder, with speed in multiple GB/s per core, typically reaching RAM speed limits on multi-core systems [lz4].

LZ4 is often used to compress large files (hundreds of GBs), e.g., [Censys] uses it to archive large JSON files.
LZ4 typically reduces such JSON file to 20-50% of the original size.

## Motivation

LZ4 is very fast but major disadvantage is missing _random access_ to the compressed file.
Imagine you have to process 1TB `json.lz4` file with one JSON record per line with only limited amount of memory.
LZ4 is designed the streaming processing is possible so just choose library that supports it.

The problem is when you have processed already 800 GB and now suddenly network glitches or there is some firewall problem or system reboots. There is currently no way to return from the last point so you basically have to download the whole file again.

## Solutions

I will demonstrate the approach in Python, using [py-lz4framed] library.

### Temporal outage

[LZ4] decompressor holds decompressing state in the memory. One solution to survive network glitches without need to modify the LZ4 library is to wrap the network read with the object transparently reconnecting to the data source on error.

In this way LZ4 decompressor is fed with the uninterrupted data stream. When error on read occurs the wrapping object
blocks, retries to connect and reads more data. When more data is loaded it is returned to the decompressor. So
even if there is 30 minutes blackout the read will resume.

I've implemented this solution here: [ReconnectingLinkInputObject](https://github.com/ph4r05/codesign-analysis/blob/9d557e7eeeeb0c461d107881278160c85a04cd56/codesign/input_obj.py#L410).
It uses `requests` library to open the network connection and `read()` from it. If there is an exception or unexpected end of data it back-offs, reconnects and uses `Range` HTTP header to request the following part of the data.

Disadvantage of this approach is in-memory only state. When system reboots the state is lost and you have to start from
the beginning.

## Modifications

- TODO: lz4 frame format
- TODO: lz4 block format
- TODO: lz4 decompressor state
- TODO: lz4 optimal breakdown (dictionaries, not linked), max block size on frame.

```c
struct LZ4F_dctx_s {
    LZ4F_frameInfo_t frameInfo;  // basic frame info
    U32    version;
    U32    dStage;
    U64    frameRemainingSize;
    size_t maxBlockSize;    // max 4MB block
    size_t maxBufferSize;   // maxBlockSize + 128 kB in linked mode
    BYTE*  tmpIn;           // memory buffer for input
    size_t tmpInSize;
    size_t tmpInTarget;
    BYTE*  tmpOutBuffer;    // memory buffer for output
    const BYTE*  dict;      // decompress dictionary, 64 kB, offset to tmpOutBuffer
    size_t dictSize;
    BYTE*  tmpOut;          // output buffer, offset to tmpOutBuffer
    size_t tmpOutSize;      // size of the data in the output buffer
    size_t tmpOutStart;     // start of the data in the output buffer
    XXH32_state_t xxh;      // checksum state 48 B (frame-wise)
    BYTE   header[16];      // frame header copy
};  /* typedef'd to LZ4F_dctx in lz4frame.h */
```

As you can see the state has quite simple structure. There are 2 memory buffers - input, output (dict), checksum state
and frame header info. So if this complete state is serialized it corresponds to the decompressor snapshot. So
after the decompressor loads serialized state it can continue.


### State marshalling

In order to recover also from program crashes you can marshal / serialize
the decompressor context to the (byte) string which can be later
unmarshalled / deserialized and continue from that point. Marshalled state
can be stored e.g., to a file. More in test `test_decompressor_fp_marshalling`.

### Random access archive

Situation: 800 GB LZ4 encrypted file. You want random access the file
 so it can be map/reduced or processed in parallel from different offsets.

Marshalled decompressor state takes only the required
amount of memory. If the state dump is performed on the block boundaries
(i.e., when the size hint from the previous call was provided by the input stream)
 the marhsalled size would be only 184 B, in the best case scenario, 66 kB in the worse case -
 when LZ4 file is using linked mode.

Anyway, when state marshalling returns this small state the application
can build a meta file, the mapping: position in the input stream -> decompressor context.
With this meta file a new decompressor can jump to the particular checkpoint.



[lz4]: https://github.com/lz4/lz4
[Censys]: https://censys.io/
[frame-format]: https://github.com/lz4/lz4/wiki/lz4_Frame_format.md
[block-format]: https://github.com/lz4/lz4/wiki/lz4_Block_format.md
[how-lz4-works]: https://ticki.github.io/blog/how-lz4-works/
[lz4-explained]: https://fastcompression.blogspot.cz/2011/05/lz4-explained.html
[lz4-streaming-format]: https://fastcompression.blogspot.cz/2013/04/lz4-streaming-format-final.html
[py-lz4framed]: https://github.com/Iotic-Labs/py-lz4framed
