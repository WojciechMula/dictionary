# dictionary
Experiments with dictionary coding

Status: at this point, this is just a technology demo to see what might be possible.
This repository might evolve, later, into something that's useful. Ideas, contributions,
criticism, collaboration are invited. (Please don't use this code in production.)

Suppose you want to compress a large array of values with
(relatively) few distinct values. For example, maybe you have 16 distinct 64-bit
values. Only four bits are needed to store a value in the range [0,16) using
binary packing,  so if you have long arrays, it is possible to save 60 bits per value (compress
the data by a factor of 16).


We consider the following (simple) form of dictionary coding. We
have a dictionary of 64-bit values (could be pointers) stored
in an array. In the compression phase, we convert the values to indexes
and binary pack them. In the decompression phase, we
try to recover the dictionary-coded values as fast as possible.

We are going to assume that one has a recent Intel processor
for the sake of this experiment.

## Core Idea

It is tempting in dictionary coding, to first unpack the indexes to a temporary buffer
and then run through it and look-up the values in the dictionary. What if it were possible
to decode the indexes and look-up the values in the dictionary in one step?
It is possible with vector instructions as long as you have access to a ``gather``
instruction. Thankfully, recent commodity x64 processors have such an instruction.

## A word on RAM access

There is no slower processor than an idle processor waiting for the memory
subsystem.

When working with large data sets, it is tempting to decompress them from RAM
to RAM, converting gigabytes of compressed data into (many more) gigabytes of
uncompressed data.

If the purpose of compression is to keep more of the data close to the CPU,
then this is wasteful.

One should engineer applications so as to work on cache-friendly blocks. For
example, if have an array made of billions of values, instead of decoding them
all to RAM, and then reading them, it is much better to decode them in small blocks
at a time. In fact, ideally, one would prefer not to decode the data at all if possible:
working directly over the compressed data would be ideal.

If you must decode gigabytes of data to RAM or to disk, then you should expect
to be wasting enormous quantities of CPU cycles.

## Credit

Builds on work done by Eric Daniel for ``parquet-cpp``.  

## Usage

```bash
make && make test
./decodebenchmark
```

## Experimental results (Skylake, August 24th 2016)

We find that an AVX2 dictionary decoder can be more than twice as fast as a good scalar decoder
on a recent Intel processor (Skylake) for modest dictionary sizes. Even with large
dictionaries, the AVX2 gather approach is still remarkably faster. See results below. We expect results on older
Intel architectures to be less impressive because the ``vpgather`` instruction that we use was
quite slow in its early incarnations.

```bash
$ ./decodebenchmark
For this benchmark, use a recent (Skylake) Intel processor for best results.
Intel processor:  Skylake     compiler version: 5.3.0 20151204        AVX2 is available.
Using array sizes of 8388608 values or 65536 kiB.
testing with dictionary of size 2
Actual dict size: 2
        scalarcodec.uncompress(t,newbuf):  4.00 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.06 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.45 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.91 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.15 cycles per decoded value

testing with dictionary of size 4
Actual dict size: 4
        scalarcodec.uncompress(t,newbuf):  3.99 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.06 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.46 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.91 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.19 cycles per decoded value

testing with dictionary of size 8
Actual dict size: 8
        scalarcodec.uncompress(t,newbuf):  3.52 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  2.38 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.49 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.93 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.17 cycles per decoded value

testing with dictionary of size 16
Actual dict size: 16
        scalarcodec.uncompress(t,newbuf):  4.01 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.08 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.50 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.95 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.19 cycles per decoded value

testing with dictionary of size 32
Actual dict size: 32
        scalarcodec.uncompress(t,newbuf):  4.02 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.06 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.51 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.96 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.18 cycles per decoded value

testing with dictionary of size 64
Actual dict size: 64
        scalarcodec.uncompress(t,newbuf):  4.02 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.08 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.54 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.98 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.17 cycles per decoded value

testing with dictionary of size 128
Actual dict size: 128
        scalarcodec.uncompress(t,newbuf):  3.59 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  2.35 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.55 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  1.99 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.14 cycles per decoded value

testing with dictionary of size 256
Actual dict size: 256
        scalarcodec.uncompress(t,newbuf):  4.03 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.10 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.55 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.00 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.22 cycles per decoded value

testing with dictionary of size 512
Actual dict size: 512
        scalarcodec.uncompress(t,newbuf):  4.04 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.11 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.55 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.01 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.20 cycles per decoded value

testing with dictionary of size 1024
Actual dict size: 1024
        scalarcodec.uncompress(t,newbuf):  4.04 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.11 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.57 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.04 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.18 cycles per decoded value

testing with dictionary of size 2048
Actual dict size: 2048
        scalarcodec.uncompress(t,newbuf):  4.08 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.15 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.67 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.05 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.22 cycles per decoded value

testing with dictionary of size 4096
Actual dict size: 4096
        scalarcodec.uncompress(t,newbuf):  4.14 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.33 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.69 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.12 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.32 cycles per decoded value

testing with dictionary of size 8192
Actual dict size: 8192
        scalarcodec.uncompress(t,newbuf):  4.35 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.65 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  3.85 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.28 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  1.67 cycles per decoded value

testing with dictionary of size 16384
Actual dict size: 16384
        scalarcodec.uncompress(t,newbuf):  4.51 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.95 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  4.07 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  2.55 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  2.12 cycles per decoded value

testing with dictionary of size 32768
Actual dict size: 32768
        scalarcodec.uncompress(t,newbuf):  4.88 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  3.84 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  4.89 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  3.52 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  3.02 cycles per decoded value

testing with dictionary of size 65536
Actual dict size: 65536
        scalarcodec.uncompress(t,newbuf):  7.14 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  5.47 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  6.68 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  5.18 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  4.53 cycles per decoded value

testing with dictionary of size 131072
Actual dict size: 131072
        scalarcodec.uncompress(t,newbuf):  7.96 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  6.05 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  7.53 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  6.01 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  5.43 cycles per decoded value

testing with dictionary of size 262144
Actual dict size: 262144
        scalarcodec.uncompress(t,newbuf):  8.30 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  6.35 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  8.08 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  6.46 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  5.66 cycles per decoded value

testing with dictionary of size 524288
Actual dict size: 524288
        scalarcodec.uncompress(t,newbuf):  8.48 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  6.39 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  8.09 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  6.44 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  5.83 cycles per decoded value

testing with dictionary of size 1048576
Actual dict size: 1048235
        scalarcodec.uncompress(t,newbuf):  11.85 cycles per decoded value
   decodetocache(&sc, &t,newbuf,bufsize):  10.53 cycles per decoded value
           avxcodec.uncompress(t,newbuf):  11.65 cycles per decoded value
  AVXDictCODEC::fastuncompress(t,newbuf):  8.47 cycles per decoded value
     AVXdecodetocache(&t,newbuf,bufsize):  8.07 cycles per decoded value
```

## Limitations
- For simplicity, we assume that the dictionary is made of 64-bit words. It is hard-coded in the code, but not a fundamental limitation: the code would be faster with smaller words.
- This code is not meant to be use in production. It is a demo.
- This code makes up its own convenient format. It is not meant to plug as-is into an existing framework.
- We assume that the arrays are large. If you have tiny arrays... well...
- We effectively measure steady-state throughput. So we ignore costs such as loading up the dictionary in CPU cache.
