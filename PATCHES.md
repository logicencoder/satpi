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

## Bug: DiSEqC broken on FBC hardware (Vu+ Duo 4K SE)

Upstream SatPI could not drive a DiSEqC switch on FBC tuner inputs. Three
independent root causes, all fixed by matching the proven minisatip
implementation (`minisatip/src/dvb.cpp`, `adapter.cpp`):

### Root cause 1 — DiSEqC sent to the wrong frontend fd

`FBC::getFileDescriptorOfRootTuner()` built the root path from a fixed
`_offset` (`fbcSetID * 8`), so a request on frontend9 sent DiSEqC to
frontend8. It also replaced only the last character of the path, so for
two-digit children (frontend10+) `fePath.replace(end-1, end, N)` produced
e.g. `frontend19`.

Fix: use `_fbcConnect` (the real root read from `fbc_connect` proc) and
replace the whole `frontendN` suffix.

### Root cause 2 — fd lifetime / shared file context

Upstream opened a *fresh* fd to the root for DiSEqC, sent the command and
closed it. minisatip instead keeps `ad->fe` open for the adapter lifetime
and sends DiSEqC on `master->fe` — the driver's per-open file context
(LNB voltage/tone state) stays alive.

Fix: `getFileDescriptorOfRootTuner()` now scans `/proc/self/fd` and
`dup()`s the already-open fd of the root frontend; if none is open it
opens once and caches the base fd in `rootFdCache` for the process
lifetime. The caller closes only the dup.

`FBC::doSendDiSEqcViaRootTuner()` now returns true only for **child**
tuners (`!_fbcRoot`) — a root sends DiSEqC on its own fd.

### Root cause 3 — FBC link state not applied at tune time

Upstream wrote `fbc_link`/`fbc_connect` only when the XML config changed.
After a reboot or an Enigma2 run the proc values could be stale/reset.

Fix: `FBC::applyFBCConfiguration()` is called at the start of
`DVBS::tune()` — root: `fbc_link=0`, `fbc_connect=<own index>`; child:
`fbc_link=<linked?>`, `fbc_connect=<root index>`.

### DiSEqC wire sequence aligned with minisatip

`DiSEqc::sendDiseqcMasterCommand()` now takes `targetVoltage` + `hiband`
and does:

```
FE_SET_TONE OFF -> FE_SET_VOLTAGE (13V/18V by polarization) ->
FE_DISEQC_SEND_MASTER_CMD -> mini-burst -> FE_SET_TONE (hiband)
```

instead of forcing 18V before and 13V after the command. Default delays
changed 35/40 ms -> 15/54 ms (minisatip `diseqc_timing`). Mini-burst is
selected by position parity `(src & 1)` like minisatip
(`pos & 1 ? MiniB : MiniA`), not the upstream `src & 0x80`.

### Note on the committed-switch position byte

`DVBS::tune()` passes `getDiSEqcSource() - 1`, i.e. `src` inside
`sendDiseqc()` is **already 0-based**. The committed position encoding is
therefore `(src & 0x03) << 2` (A=0xF0, B=0xF4, C=0xF8, D=0xFC) — identical
to minisatip's `0xf0 | (pos << 2 & 0x0c)`. Do not subtract 1 again.

### `fe=N` HTTP parameter is 1-based

`StreamManager::findFrontendID()` maps `fe=N` to `_streamVector[N-1]`,
i.e. `fe=10` is `/dev/dvb/adapter0/frontend9`. Keep this in mind when
testing — `fe=9` addresses frontend8.

### Empirical verification (Vu+ Duo 4K SE, OpenPLi 9.2)

frontend1 (4-port DiSEqC A/B/C/D switch on Slot A input B):

| src | pos | sat   | transponder    | result                |
|-----|-----|-------|----------------|-----------------------|
| 1   | A   | 23.5E | 11739V/29900   | 2044 real TS pkts, 0x1F |
| 3   | C   | 19.2E | 11347V/22000   | 6139 real TS pkts, 0x1F |
| 4   | D   | 13E   | 12188V/27500   | 4095 real TS pkts, 0x1F |

Wire bytes observed in the log: `0xF1` (A+hiband), `0xF8` (C),
`0xFD` (D+hiband). Child frontend10 correctly routes DiSEqC through
root frontend9 and locks.

## Bug: kernel oops + no data — NEXUS (bcm7335) demux quirks

On the Vu+ NEXUS (Broadcom bcm7335) driver the stock SatPI demux path both
crashed the kernel and delivered no TS data. Empirically mapped behaviour
(LD_PRELOAD ioctl tracer + standalone reproducers, all on OpenPLi 9.2):

- `DMX_SET_SOURCE` must be issued on the **same fd** that owns the filters.
  A temp-fd SET_SOURCE then close (old code) leaves the filter fd
  unbound -> `DMX_START` inside `dvb_dmxdev_filter_start` derefs a NULL
  recpump -> kernel oops. Fixed: per-frontend `demuxN` fd is kept open and
  owns source + buffer + all filters.
- The demux must be opened **~450 ms after `FE_SET_PROPERTY`** (same
  moment the demod reports lock). Opening it before tune or seconds later
  binds to nothing.
- Data starts flowing **~7 s after tune** on this driver regardless.
- A lone `DMX_SET_PES_FILTER` never delivers; the recpump only starts once
  **>= 2 `DMX_ADD_PID`** channels exist.
- **Mid-stream `DMX_ADD_PID` works** (new pid first packet ~5 s later) as
  long as the total stays under the driver's channel limit (~29).
  Exceeding it (~34) makes the whole ADD batch deliver nothing.
- A second `demuxN` fd on the same frontend never delivers, `dvrN` reads
  return nothing, `DMX_OUT_TAP` output is dead, and pid `0x2000`
  ("all pids") is just an ordinary pid here -- and crashed the driver
  outright when sent via `DMX_SET_PES_FILTER` (the original oops).
- `DMX_ADD_PID` takes `int*`, not `uint16_t*` -- the driver writes the
  channel handle back into the arg (seen as `0x280000|pid` in traces).

### What the code now does

- `openPid()` (Frontend.cpp): keeps `demuxN` fd, `DMX_SET_SOURCE` on it,
  3 MB `DMX_SET_BUFFER_SIZE`, first pid via `DMX_SET_PES_FILTER`, rest via
  `DMX_ADD_PID`. Pids `>= 0x2000` are skipped entirely (kernel guard).
- `pids=all` emulation (`Filter.cpp`): never sends 8192 to hardware;
  walks PAT -> PMT -> ES and auto-opens discovered pids via mid-stream
  `DMX_ADD_PID`. Capped: `EMU_PMT_BUDGET=8` PMT pids, `EMU_PID_BUDGET=24`
  total, so the session stays under the ~29-channel limit and video/audio
  of the first services really flow.
- `readTSPackets()` calls `updatePIDFilters()` whenever the pid table
  changed mid-stream (newly discovered pids get `ADD_PID` immediately --
  no demux rebuild; an earlier close/reopen approach killed the session).
- Failed pid opens are marked closed instead of retry-storming ioctls.

Empirical `pids=all` result on 11739V (23.5E): **201k packets, 26 pids**
including video (pid 566 = 48k pkts) + audio/ES -- a real partial TS.

## Notes on this installation (Vu+ Duo 4K SE, OpenPLi 9.2)

- Binary deployed as `/usr/bin/satpi_patched`, started by
  `/etc/init.d/satpi` (rc symlinks in rcS.d/rc2-5.d).
- Tuner split: Enigma2 owns Slot A (frontend0-7, incl. the 4-port DiSEqC
  switch on frontend1); SatPI owns Slot B (`/etc/satpi/SatPI.xml` has
  streams 1-8 `enable=false`). Enigma2 is not stopped by the init script.
- `diseqcType` in `SatPI.xml`: `0`=DiSEqc Switch, `3`=Lnb. Configure per
  stream; the C++ default stays `Lnb` like upstream.
- Source tarball of this tree: `/etc/satpi/satpi_src_v2_working.tar.gz`
  on the box.
