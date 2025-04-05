---
title: The Quiet Joy of Simplicity (RFC 868)
date: Sat, 05 Apr 2025 12:34:28 -0400
draft: false
tags: ['linux', 'bash']
---

I recently discovered the Time protocol, as defined by [RFC 868](https://www.rfc-editor.org/rfc/rfc868).
This delightfully simple RFC, if implemented, returns an unsigned 32-bit integer as bytes, then disconnects.
That integer represents the number of seconds since 1900-01-01 00:00:00+00:00, the epoch stated in the RFC.

That's it. That's the entire thing. NIST maintains [this delightfully Web 1.0 page](https://tf.nist.gov/tf-cgi/servers.cgi)
where they state that they service Time requests on port 37 (though they'd prefer you to use NTP). This, in fact, does work:

```bash
❯ nc -w1 time.nist.gov 37 | god -An -tu4 -N4 --endian=big | awk '{print $1 - 2208988800}' | xargs -I{} gdate -d @{}

Sat Apr  5 12:38:02 EDT 2025
```

This is, in order:

1. Open a connection to the server on port 37 with a short timeout.
2. Pipe the output to GNU od, specifying no radix in the offset, unsigned 4-byte integer, reading exactly 4 bytes, in big-endian order.
3. Pipe the output to awk, printing stdin minus an integer representing the number of seconds between their epoch and Unix's.
4. Pipe that into GNU date, informing it that it is to treat the input as a Unix timestamp.

Or, in Python:

```python
import socket
import struct
import time

def get_time():
    HOST_PAIR = ("time.nist.gov", 37)
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect(HOST_PAIR)
        t = struct.unpack("!I", s.recv(4))
    print(time.ctime(t[0] - 2208988800))
```

Example output:

```python
>>> get_time()
Sat Apr  5 13:07:06 2025
```

You can also trivially implement a (terrible) server for this:

```python
import socket
from datetime import datetime, timezone


def serve():
    EPOCH = datetime.fromisoformat("1900-01-01 00:00:00+00:00")
    HOST_PAIR = (
        "",
        12345,
    )  # or 37 to be correct, but then it's privileged, so run it with sudo
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(HOST_PAIR)
        s.listen(1)  # N = allowed backlog of connections
        while True:
            conn, addr = s.accept()
            with conn:
                cur_dt = datetime.now(timezone.utc)
                delta_dt_sec = int((cur_dt - EPOCH).total_seconds())
                data = delta_dt_sec.to_bytes(
                    length=4, byteorder="big", signed=False
                )  # the last two are defaults FWIW
                try:
                    conn.sendall(data)
                    print(
                        f"INFO: {cur_dt.ctime()}: sent {delta_dt_sec} as {data} to {addr}"
                    )
                except socket.error:
                    print(f"ERROR: failed to send datetime to {addr}")
```

Example output:

```
>>> serve()
Sat Apr  5 15:29:06 2025: sent 3952855746 as b'\xeb\x9b\xca\xc2' to ('127.0.0.1', 59314)
Sat Apr  5 15:30:10 2025: sent 3952855810 as b'\xeb\x9b\xcb\x02' to ('127.0.0.1', 59315)
Sat Apr  5 15:47:36 2025: sent 3952856856 as b'\xeb\x9b\xcf\x18' to ('127.0.0.1', 59334)
Sat Apr  5 15:47:50 2025: sent 3952856870 as b'\xeb\x9b\xcf&' to ('127.0.0.1', 59335)
Sat Apr  5 15:47:53 2025: sent 3952856873 as b'\xeb\x9b\xcf)' to ('127.0.0.1', 59336)
```

NOTE: I've occasionally found that requests to `time.nist.gov` fail, presumably because one of the servers it load-balances to
(`time-f-wwv.nist.gov`) _only_ services NTP. Feel free to create your own list of working servers located close to you.
