# SatPI patches (Logic Encoder fork)

Base: upstream `Barracuda09/SATPI` commit `09df7402870b` (same commit as the
OpenPLi 9.2 `satpi git2+09df740287-r0.0` package). License: GPLv2 (unchanged,
see `COPYING`).

## Bug: SIGSEGV on whitespace-only RTSP keepalive

`SocketClient::getMethod()` and two other methods index `headers[0]` without a
bounds check. When a client sends a whitespace-only message (`\r\n\r\n`,
4 bytes — a normal RTSP keep-alive sent by e.g. DVBViewer ~1 minute into a
stream), `StringConverter::split` drops all empty tokens, `getHeaders()`
returns an empty vector, and `headers[0]` dereferences `_M_start == 0` →
`std::string::empty(this=0x0)` → SIGSEGV → whole process dies.

GDB backtrace captured on Vu+ Duo 4K SE:

```
Thread "RTSP Server" received signal SIGSEGV
std::string::empty (this=0x0)
SocketClient::getMethod()      src/socket/SocketClient.h
HttpcServer::process           src/HttpcServer.cpp
TcpSocket::poll                dataSize=4
RtspServer::threadEntry
```

## Patch

`src/socket/SocketClient.h`:

1. `getMethod()` — return early when `headers.size() == 0`; bound both
   iterators by `it != line.end()` (also fixes a potential over-read on
   space-less binary junk).
2. `getRequestedFile()` — same empty-headers guard.
3. `getTransportParameters()` — same guard, returns an empty
   `TransportParamVector`.

No behavior change for valid HTTP/RTSP requests.

## Build (cross-compile for Vu+ armv7, OpenPLi)

```bash
# Bootlin prebuilt toolchain (no root needed):
wget https://toolchains.bootlin.com/downloads/releases/toolchains/armv7-eabihf/tarballs/armv7-eabihf--glibc--stable-2020.08-1.tar.bz2
tar -xjf armv7-eabihf--glibc--stable-2020.08-1.tar.bz2
export PATH=$PWD/armv7-eabihf--glibc--stable-2020.08-1/bin:$PATH

make CXX=arm-buildroot-linux-gnueabihf-g++ \
     CPU_FLAGS="-march=armv7-a -mfpu=neon -mfloat-abi=hard" \
     LDFLAGS="-pthread -lrt -static-libstdc++ -static-libgcc" -j4
# output: ./satpi  (ELF 32-bit ARM, ~2.3 MB)
```

Deploy on the box:

```bash
scp satpi root@BOX:/usr/bin/satpi.bin.new
ssh root@BOX '
  /etc/init.d/satpi stop; killall -9 satpi.bin
  cp /usr/bin/satpi.bin /usr/bin/satpi.bin.orig
  mv /usr/bin/satpi.bin.new /usr/bin/satpi.bin
  /etc/init.d/satpi start'
```

## Regression test for the crash

```bash
python3 -c 'import socket,time; s=socket.socket(); \
  s.connect(("BOX_IP",554)); s.send(b"\r\n\r\n"); time.sleep(1); s.close()'
ps | grep satpi.bin   # must stay alive (old binary died here)
```

## Notes on this installation (Vu+ Duo 4K SE, OpenPLi 9.2)

- `/usr/bin/satpi` is a Python wrapper that ignores `SIGPIPE` and execs
  `/usr/bin/satpi.bin` (separate crash source — keep it).
- `/etc/init.d/satpi` must use `killall satpi.bin` (not `killall satpi`),
  otherwise stopped instances survive and the next instance cannot bind
  port 8875.
- SatPI frontends in `/etc/satpi/SatPI.xml`: only `frontend9` (physical
  Tuner B lower input) enabled. FBC children `frontend2-7` belong to the
  Enigma2-owned input and children `frontend10-15` cannot send DiSEqC on
  this driver — all must stay disabled or clients get
  `503 No-More: frontends` and leaked `attached=yes` frontends.
- `waitOnLockTimeout` raised to 3000 ms: DiSEqC transmit on this input
  logs a benign `bcm7335_send_diseqc_msg` timeout (~1.1 s) before tuning.
- Cron watchdog in root crontab restarts SatPI if it ever stops:
  `* * * * * /etc/init.d/satpi status >/dev/null 2>&1 || /etc/init.d/satpi start`
