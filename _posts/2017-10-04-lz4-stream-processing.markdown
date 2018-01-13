---
layout: post
title:  "LZ4 streamed processing of large files"
date:   2017-10-04 22:10:39 +0200
categories: blog
excerpt_separator: <!-- more -->

---

Processing terabyte large LZ4 files in a streaming fashion with resumption and random access.

<!-- more -->

## Intro

LZ4 is lossless compression algorithm, providing compression speed at 400 MB/s per core, scalable with multi-core CPU. It features an extremely fast decoder, with speed of multiple GB/s per core, typically reaching RAM speed limits on multi-core systems [lz4].

LZ4 is often used to compress large files (hundreds of GBs), e.g., [Censys] uses it to archive Internet-wide
 TLS scans - large JSON files.
LZ4 typically reduces such JSON file to 20-50% of the original size.

[![Censys IPv4 scan snapshot](/static/lz4/censys_ipv4.png)](/static/lz4/censys_ipv4.png)

`curl --head` returns `Content-Length: 242919507913` which is roughly 226.24 GB reducing the file size to 23% of the original size. JSON is quite redundant (field names) making it a good subject to compression.

## Motivation

Imagine you have to process 1TB `json.lz4` file with one JSON record per line with the limited amount of memory and disk storage. You definitely cannot download the file or load it to the memory.
Fortunately, LZ4 is designed the streaming processing is possible so you only need to choose a library that supports it.

There is a `lz4` package available on RHEL/Ubuntu to work with LZ4 archives, e.g., fetching the first record from the compressed json can be done like this:

```bash
$> curl -s -L 'https://ph4r05.deadcode.me/static/lz4/certificates.20171002T020001.15.json.lz4' | lz4 -d -c - | head -n 1
{"fingerprint_sha256":"f9000003ebc6a586e0076ee11f446cf2a844461bfee6e9d2735fabfd99aba231","raw":"MIIDUTCCAjmgAwIBAgIFFIdXdAkwDQYJKoZIhvcNAQELBQAwUjELMAkG ....
```

A major disadvantage of LZ4 is lack of _random access_ to the compressed file.

The problem is when you have processed already 800 GB and now suddenly network glitches or there is some firewall problem or system reboots. There is currently no way to return from the last point so you basically have to download the whole file again.

I will demonstrate solutions in Python, using [py-lz4framed] library, forked version
of the [py-lz4framed original] repository with few my additions which I explain in this post.

## Stream processing

LZ4 decompressor accepts a file-like object with `read()` method. It can be e.g., locally opened file or a network socket. The naive way to read a remote file in a streaming fashion is to open the remote connection and pass the socket object to the decompressor like this:

```python
import requests, lz4framed
url = 'https://ph4r05.deadcode.me/static/lz4/certificates.20171002T020001.15.json.lz4'
r = requests.get(url, stream=True, allow_redirects=True)
for idx, chunk in enumerate(lz4framed.Decompressor(r.raw)):
    print(chunk)
```

If there is a timeout/connection reset or network glitch either the exception is thrown or empty data is returned from the `read()` which causes decompressor to fail.

## Temporal outage - reconnect wrapper

[LZ4] decompressor holds decompressing state internally in the memory. One solution to survive network glitches without a need to modify the LZ4 library is to wrap the network file-like object with the object transparently reconnecting to the data source on error.

In this way, LZ4 decompressor is fed with the uninterrupted data stream. When an error on reading occurs the wrapping object
blocks, retries to connect and reads more data. When more data is loaded it is returned to the decompressor. So
even if there is 30 minutes blackout the read will resume.

I've implemented this solution here: [ReconnectingLinkInputObject](https://github.com/ph4r05/input_objects/blob/02c4e2b031410ffe35721c33f8abe8be92a07a91/input_objects/input_obj.py#L410).
It wraps `requests` library to open the network connection and read from it. If there is an exception or unexpected end of data it back-offs, reconnects and uses `Range` HTTP header to request unseen part of the data from the server.

```python
import requests, lz4framed, input_obj
url = 'https://ph4r05.deadcode.me/static/lz4/certificates.20171002T020001.15.json.lz4'
iobj = input_obj.ReconnectingLinkInputObject(url=url, timeout=5*60, max_reconnects=1000)
for idx, chunk in enumerate(lz4framed.Decompressor(iobj)):
    print(chunk)
```

A disadvantage of this approach is in-memory only state. When system reboots the state is lost and you have to start from
the beginning.

To use this library you can either go for GitHub repository [https://github.com/ph4r05/input_objects](https://github.com/ph4r05/input_objects) 
or install it via pip:

```bash
pip install input_objects
```

## Advanced approach - state backup

In order to be able to resume after OS crash we need to backup the decompressor state somehow.

## LZ4 Format

LZ4 format consists of frames (top level format) which contains LZ4 blocks.

The frame format encapsulating the data is the following:

[![Frame format](/static/lz4/frame-format.svg)](/static/lz4/frame-format.svg)

- Magic bytes and end mark are fixed 4 bytes marks
- Frame descriptor defines basic setting of the LZ4 algorithm - explained below.
- LZ4 data blocks are present multiple times
- The checksum is [xxHash] of the data blocks - if enabled in the frame descriptor.

Usually, there is only one frame encapsulating the whole data in the LZ4 file.
The frame format is described in detail: [frame-format].

### Frame descriptor

Frame descriptor contains basic flags defining the processing of the LZ4 archive.

[![Frame descriptor](/static/lz4/lz4-descriptor.svg)](/static/lz4/lz4-descriptor.svg)

- Block independence: 1 means blocks are independent and can be decoded separately (i.e., the dictionary is not shared among).
0 means blocks are dependent on each other (up to the LZ4 window size - 64 KB).

- Block checksum: 1 means each block in the frame is followed by 4B [xxHash] checksum.

- Content size: 1 means uncompressed content size is stored in the frame descriptor.

- Block Maximum Size: maximum size of an uncompressed block, helps with allocating memory. Options: 64 KB, 256 KB, 1 MB, 4 MB.

### LZ4 data blocks

Data blocks stored in the frame have the following structure:

[![LZ4 data blocks](/static/lz4/lz4-block.svg)](/static/lz4/lz4-block.svg)

Data part contains the LZ4 compressed data, specified in [block-format].
For the scope of this blog post the internal structure of the block is not
important as we work with them as opaque data units processed by low-level LZ4 engine.

### Decompressor state

Here is the main state of the decompressor in C.
It is here just for demonstration purposes to demonstrate the complexity of the
state (and its size). You don't have to understand the fields meaning.

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

As you can see the state has quite simple structure. There are 2 memory buffers - input, output; (dict), checksum state
and frame header info. So if this complete state is serialized it corresponds to the decompressor snapshot at a particular
position in the LZ4 archive. 
So after the decompressor loads serialized state it can continue from the given position.

This state marshaling extension was implemented in the LZ4 library. 
User can periodically call state marshaling method and build a dictionary
with snapshots of decompressor snapshots at given reading position of the stream.

Example of the snapshot/checkpoint map in reading positions 10, 20, 30 MB:

```json
[
  {
      "pos": 10485760,
      "state": "Abz01fc0ad.........."
  },
  {
      "pos": 20971520,
      "state": "Abz01fc1ad.........."
  },
  {
      "pos": 31457280,
      "state": "Abz01fc1ad.........."
  }
]
``` 

Such `position -> decompressor state` state mapping file can be built on the fly when reading the file for the first time.

When the server crashes or the process is corrupted the mapping file is read, the last entry is located so we get
\\((\text{pos}_l, \text{state}_l)\\). 

The file is then opened at the position \\(\text{pos}_l\\), decompressor state 
\\(\text{state}_l\\) is resumed and the process can continue.

The file reading from the given position can be done with seeking or by using 
HTTP Range headers if the file is read from a remote server.

Take a look at the test [`test_decompressor_fp_marshalling`](https://github.com/ph4r05/py-lz4framed/blob/00e3984d6de98fa9562f15070a6982deeae4e215/test.py#L481)
for a detailed example of state marshaling and unmarshalling.

### Random access archive

Situation: 800 GB LZ4 encrypted file. You want random access the file
 so it can be map/reduced or processed in parallel from different offsets.
 
If you build the LZ4 `position -> decompressor state` mapping file 
other processes can use this file to jump to the checkpoints in the file 
without the need to read the file from the beginning. 

This simple way enables you to random access LZ4 archive at cost of building
the mapping file (one extra read).

You can try building your own mapping file for the LZ4 archive using
my GitHub project [lz4-checkpoints].

### State size

A position of a decompressor in the LZ4 archive being read heavily influences the size of the serialized
decompressor state. 

When you marshall the decompressor state in the middle of a block the
decompressor state need to contain the whole decompressing state so
it can continue processing the block after resume. It means it can take up
to 8 MB (if the block size is 4 MB), give or take some additional data like a checksum.

Naively building a mapping file is then space ineffective. But this can be improved
significantly.  

If the frame format is using non-linked blocks (each block is independent) 
then the state size of the decompressor at the position between blocks is minimal.

There is no need to store the block buffers after the block reading has finished. The only
size that needs to be stored is the content checksum [xxHash] state so decompressor 
can compute the checksum of all stored blocks when finished. 

State serialized at the block boundaries takes only 184 Bytes.

The LZ4 decompressing library has an API giving you so-called size hints after each read. 
Size hint is the preferred size of the input archive chunk to process in the next read 
call so the memory allocation in the decompressor is optimal. 

In other words, size hint is a number of bytes to read from the input archive so 
the block is processed in one call. When reading the archive in this way the
file is read and processed by blocks, which is more effective.

Using the size hints when reading and processing input LZ4 archive by chunks
determines the block boundaries in the input archive. When doing the checkpointing
we store the decompressor state exactly at those positions to minimize 
the size of the mapping file.

When processing the LZ4 archive you may also find handy to store one more
information to the checkpoint - position in the *uncompressed* data stream
(data produced by decompressor). This may be useful when jumping in the 
data stream - depends on the structure of the compressed file.

```json
[
  {
      "pos": 10485760,
      "upos": 24117248,
      "state": "Abz01fc0ad.........."
  },
  {
      "pos": 20971520,
      "upos": 61865984,
      "state": "Abz01fc1ad.........."
  },
  {
      "pos": 31457280,
      "upos": 81788928,
      "state": "Abz01fc1ad.........."
  }
]
``` 

### Uncompressed Data format

In our case, the LZ4 contains line-separated JSON records which are ideal for this kind of random access.

It usually happens the block boundary is not precisely at the position in the decompressed JSON
file where the new record stars (new line) so when jumping to the checkpoint you usually occur in the middle
of the JSON record.

[lz4]: https://github.com/lz4/lz4
[Censys]: https://censys.io/
[frame-format]: https://github.com/lz4/lz4/wiki/lz4_Frame_format.md
[block-format]: https://github.com/lz4/lz4/wiki/lz4_Block_format.md
[how-lz4-works]: https://ticki.github.io/blog/how-lz4-works/
[lz4-explained]: https://fastcompression.blogspot.cz/2011/05/lz4-explained.html
[lz4-streaming-format]: https://fastcompression.blogspot.cz/2013/04/lz4-streaming-format-final.html
[py-lz4framed]: https://github.com/ph4r05/py-lz4framed
[py-lz4framed original]: https://github.com/Iotic-Labs/py-lz4framed
[lz4-checkpoints]: https://github.com/ph4r05/lz4-checkpoints
[xxHash]: https://github.com/Cyan4973/xxHash
