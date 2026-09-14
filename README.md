# SatPI — Logic Encoder fork

An SAT>IP server for Linux (DVB-S/S2, DVB-T/T2, DVB-C), forked from
[Barracuda09/SATPI](https://github.com/Barracuda09/SATPI) — all credit for the
original project goes to its author.

This fork exists to make SatPI actually work on the **Vu+ Duo 4K SE**
(Broadcom bcm7335 / NEXUS, dual FBC DVB-S2) — a box where upstream delivers
no data at all on the second tuner bank and cannot switch DiSEqC
([upstream issue #210](https://github.com/Barracuda09/SATPI/issues/210),
open since 2024). Verified end-to-end on OpenPLi 9.2 with **DVBViewer**
switching three orbital positions (23.5E / 13E / 19.2E) while Enigma2 runs
alongside.

## What this fork fixes

| Area | Fix |
|------|-----|
| `socket/SocketClient.h` | SIGSEGV on whitespace-only RTSP keepalive (`headers[0]` on empty vector) |
| `main.cpp` | Ignore `SIGPIPE` — a client closing TCP mid-write no longer kills the daemon |
| `input/dvb/Frontend.cpp`, `mpegts/Filter.cpp` | NEXUS demux model: same-fd `DMX_SET_SOURCE`, mid-stream `ADD_PID`, safe `pids=all` emulation |
| `input/dvb/delivery/` | DiSEqC + FBC: child tuners route DiSEqC through `dup()` of the already-open root frontend fd (minisatip parity), correct tone→voltage→cmd→mini-burst ordering |
| `TransportParamVector.cpp`, `FrontendData.cpp` | Case-insensitive SAT>IP transport values (`pol`, `msys`, `plts`, `fec`, `ro`, `mtype`, …) |
| `Frontend.cpp`, `FrontendData.h/.cpp` | On lock timeout with explicit hint params (`plts`/`fec`/`ro`), retune once with them relaxed to AUTO — clients like DVBViewer forward stale channel-list values (e.g. `plts=off` on pilots-on transponders) that can never lock |

## Vu+ Duo 4K SE deployment

- Cross-build with an ARM toolchain (e.g. `arm-gnu-toolchain-13.3`) and link
  statically: `-static -static-libstdc++ -static-libgcc` — the box's glibc is
  older than the toolchain's.
- Enable only the FBC **root** frontend of the bank you want — on bcm7335 the
  FBC children cannot emit DiSEqC (`bcm7335_send_diseqc_msg` times out on the
  child fd).
- Start satpi with `--iface-name eth0` so the SSDP `LOCATION` advertises a
  usable address instead of `0.0.0.0`.
- Reference config and init script:
  [`docs/SatPI.xml.vu-dueo4kse`](docs/SatPI.xml.vu-dueo4kse),
  [`docs/satpi.initd.vu-duo4kse`](docs/satpi.initd.vu-duo4kse).
- In Enigma2, mark the SatPI-owned tuners `configMode=nothing` so both worlds
  never open the same frontend.

The same binary should run on other images using the same driver stack
(OpenATV, VTi) — it is statically linked and only needs `/dev/dvb` plus the
XML config.

## Quick start

```sh
git clone https://github.com/logicencoder/satpi.git
cd satpi
make
./satpi --help
./satpi            # needs privilege for tcp/udp port 554
```

- Web interface: `http://<box-ip>:8875` (live frontend status, config editor)
- Device description (SSDP): `http://<box-ip>:8875/desc.xml`
- Build variants: `make debug`, `make LIBDVBCSA=yes` (OSCam/dvbapi),
  `make ENIGMA=yes` (Enigma2 toolchain), `make non-c++17` (old toolchains)

For full upstream documentation see the
[SatPI wiki](https://github.com/Barracuda09/SATPI/wiki).

## Features (from upstream)

- RTP/AVP, RTP/AVP/TCP and plain HTTP streaming
- DVB-S(2) requests transformed to DVB-C/T outputs
- Channel decryption via the DVB-API protocol with OSCam (dvbcsa)
- Virtual tuners: FILE, STREAMER (multicast/unicast), CHILDPIPE inputs
- Works with Tvheadend, DVBViewer, VDR, VLC, Elgato Sat>IP, satip-client

## License & credit

GPL-2.0, same as upstream. Original project and author:
[Barracuda09/SATPI](https://github.com/Barracuda09/SATPI) — if SatPI is
useful to you, consider donating to the original author via the sponsor
button on the upstream repo.

Fork maintained by **Logic Encoder** — https://logicencoder.com
