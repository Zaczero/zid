# ZID

[![PyPI - Python Version](https://shields.monicz.dev/pypi/pyversions/zid)](https://pypi.org/project/zid)
[![Liberapay Patrons](https://shields.monicz.dev/liberapay/patrons/Zaczero?logo=liberapay&label=Patrons)](https://liberapay.com/Zaczero/)
[![GitHub Sponsors](https://shields.monicz.dev/github/sponsors/Zaczero?logo=github&label=Sponsors&color=%23db61a2)](https://github.com/sponsors/Zaczero)

**zid** is a unique identifier with nice properties:

- It behaves like a 64-bit signed integer, so it can be safely used with external software, e.g., in a database. ZIDs will never overflow into negative values.

- ZIDs are numerically sortable since the timestamp is stored in the most significant bits. Additional randomness is stored only in the least significant bits.

- The specification is very simple, reducing the potential for bugs and making ZIDs highly efficient to generate and parse. Scroll down for the installation-free copy-and-paste code - it's that short!

- CSPRNG-initialized sequence numbers enhance the privacy of the generated identifiers while remaining collision-resistant. You can generate up to 65,536 ZIDs within the same millisecond timestamp on a single machine.

## Installation

The recommended installation method is through the PyPI package manager. The project is implemented in Rust, offering excellent performance characteristics. Several pre-built binary wheels are available for Linux, macOS, and Windows, with support for both x64 and ARM architectures.

```sh
pip install zid
```

## Installation (copy & paste)

Alternatively, you can copy and paste the following code for an installation-free ZID generator. This code excludes performance optimizations and utility methods for the sake of simplicity and portability:

```py
from os import urandom
from time import time_ns

_last_time: int = -1
_last_sequence: int = -1

def zid() -> int:
    global _last_time, _last_sequence

    # UNIX timestamp in milliseconds
    time: int = time_ns() // 1_000_000
    if time > 0x7FFF_FFFF_FFFF:
        raise OverflowError('Time value is too large')

    # CSPRNG-initialized sequence numbers
    sequence: int
    if _last_time == time:
        _last_sequence = sequence = (_last_sequence + 1) & 0xFFFF
    else:
        _last_sequence = sequence = int.from_bytes(urandom(2))
        _last_time = time

    return (time << 16) | sequence
```

## Basic usage

```py
from zid import zid
zid()  # -> 112723768038396241
zid()  # -> 112723768130153517
zid()  # -> 112723768205368402

from zid import zids
zids(3)
# -> [113103096068704205, 113103096068704206, 113103096068704207]

from zid import parse_zid_timestamp
parse_zid_timestamp(112723768038396241)
# -> 1720028198828 (UNIX timestamp in milliseconds)
```

## Format specification

ZID is 64 bits long in binary. Only 63 bits are used to fit in a signed integer. The first 47 bits are a UNIX timestamp in milliseconds. The remaining 16 bits are CSPRNG-initialized sequence numbers.

```txt
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|1|2|3|4|5|6|7|8|9|A|B|C|D|E|F|0|1|2|3|4|5|6|7|8|9|A|B|C|D|E|F|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                     timestamp (31 bits)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      timestamp (16 bits)      |   random+sequence (16 bits)   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## Limitations

- Timestamps support years from 1970 to approx. 6429. To verify this, you can follow the formula *1970 + (2^47 − 1) / 1000 / (3600 * 24) / 365.25*

- If several ZIDs are generated with the same millisecond timestamp, knowing one of them will allow you to discover the others due to linearly increasing sequence numbers. Otherwise, guessing ZID values is difficult (but not impossible) due to the millisecond precision of the timestamp and the additional 16 bits of entropy. Do not rely on ZIDs alone for your security!

- You can generate up to 65,536 ZIDs within the same millisecond timestamp on a single machine. With two separate machines, you will generate on average 16,384 ZIDs each before a collision occurs within the same millisecond timestamp. With three separate machines, the average number is 10,240 ZIDs each.

- ZIDs are not strictly increasing within the same millisecond timestamp due to the possible sequence number overflow.
