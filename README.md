# SoapyBirria

SoapyBirria is a SoapySDR plugin (`driver=birria`) for
[BirriaSDR](../BirriaSDR) receivers, the MT7961/MT7972 Wi-Fi dongles that
BirriaSDR drives in I/Q capture mode. With it, any SoapySDR application can
use the dongle. This page records the setups tested on a Debian 12 desktop
with an EDUP AX3000M (`0e8d:7961`): CubicSDR locally, and an SDR++ server
with the SDR++ Android app as a client.

## What the plugin provides

| Item | Value |
| --- | --- |
| Driver key | `driver=birria`. Select a dongle by its enumeration order with `index=N` |
| Sample rates | 40 MSps divided by 1, 2, 4, 5, 8, 10, 16, 20 or 40, from 40 MSps down to 1 MSps. The dongle always runs at 40 MSps, and the plugin decimates with a host-side FIR filter |
| Stream formats | `CS16` (native) and `CF32` |
| Frequency ranges | One range per library band, from `birriasdr_get_tuning_ranges()`. Currently 2156–2610 MHz (2190 MHz floor on `0e8d:7961`), 5150–5925 MHz and 5925–7125 MHz |
| Gains | `LNA` 0–3, `LPF` 0–8 and `DIG` 0–12. These are raw codes, not dB |
| Gain mode | Automatic (the library's default) or manual. Gain codes apply only in manual mode. In automatic mode, they are stored and sent when manual mode is selected |
| Band overrides | Device arguments `bandN_min` and `bandN_max` (Hz) replace one band's bounds through `birriasdr_set_tuning_range()`, for example `band0_min=2169e6`. A missing value keeps the board's value |

Errors from tuning, sample rate changes and gain changes are logged and not
thrown, because SDR++ does not catch exceptions from these calls. A rejected
frequency leaves the receiver where it was.

Limit: live retuning cannot cross bands. To move between 2.4 GHz, 5 GHz and
6 GHz, stop the stream, retune, then start it again.

## Build

The plugin links against a built BirriaSDR checkout next to this directory.
It must be the `tuning-ranges` branch or a later one, because the plugin uses
`birriasdr_get_tuning_ranges()`.

```
cmake -S ../BirriaSDR -B ../BirriaSDR/build && cmake --build ../BirriaSDR/build
cmake -S . -B build && cmake --build build
```

Set `-DBIRRIASDR_ROOT=<path>` to use a checkout elsewhere. The plugin finds
`libbirriasdr.so` through its build RPATH, so rebuild the plugin after you
rebuild the library on a different branch.

Check that SoapySDR finds the dongle:

```
SOAPY_SDR_PLUGIN_PATH=$PWD/build SoapySDRUtil --find=driver=birria
```

Every command below sets `SOAPY_SDR_PLUGIN_PATH`. To skip that step, install
the plugin into the system module directory:

```
sudo cmake --install build
```

## USB access without root

`/etc/udev/rules.d/99-birriasdr.rules` lets the `plugdev` group open both
supported IDs:

```
# BirriaSDR receivers: let the plugdev group open them without root.
SUBSYSTEM=="usb", ATTR{idVendor}=="3574", ATTR{idProduct}=="6211", MODE="0660", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", ATTR{idProduct}=="7961", MODE="0660", GROUP="plugdev"
```

Reload the rules and replug the dongle:

```
sudo udevadm control --reload-rules && sudo udevadm trigger
```

The `mt7921u` kernel driver must not be bound to the dongle. Earlier runs
with `sudo` leave root-owned `/tmp/birriasdr-<bus>-<addr>.lock` files, and
an unprivileged run cannot open the device until they are removed.

Only one program can open the dongle at a time. Stop the SDR++ server before
you start CubicSDR, and the reverse.

## CubicSDR on the desktop

The Debian `cubicsdr` package (0.2.7) links the system SoapySDR 0.8, the same
version the plugin builds against, so no rebuild is needed.

```
SOAPY_SDR_PLUGIN_PATH=$PWD/build CubicSDR
```

To start it from a terminal that has no display variables, such as an SSH
or background shell on the KDE X11 session, set them and detach:

```
cd ~ && DISPLAY=:0 XAUTHORITY=$HOME/.Xauthority \
  SOAPY_SDR_PLUGIN_PATH=$HOME/code/playground/hardware_hacking/diysdr/SoapyBirria/build \
  setsid nohup CubicSDR >$HOME/.cache/cubicsdr.log 2>&1 </dev/null & disown
```

In the device dialog, pick "MediaTek MT7961 / MT7972" and press Start. Turn
AGC off to use the `LNA`, `LPF` and `DIG` sliders. `{LNA 2, LPF 8, DIG 0}`
is a reasonable start. Use LNA 3 at 6 GHz. CubicSDR cannot demodulate
Wi-Fi, so use it as a spectrum and waterfall view.

## SDR++ server and the Android app

The SDR++ server runs on the desktop, owns the dongle and streams to SDR++
clients on port 5259. The Debian SDR++ package does not include the SoapySDR
source module, so SDR++ 1.3.0 was built from source into `~/.local`.

### Build SDR++ with the SoapySDR source

Install SDR++'s build dependencies as its README lists them. This build also
needed `libglfw3-dev` and `librtaudio-dev`. Hardware sources that have no
libraries here are turned off.

```
D=$HOME/src/sdrpp
git clone --depth 1 https://github.com/AlexandreRouma/SDRPlusPlus $D/src
OFF=
for m in AIRSPY AIRSPYHF BLADERF HACKRF LIMESDR PLUTOSDR RTL_SDR DRAGONLABS FOBOSSDR \
         HYDRASDR RFNM PERSEUS SDRPLAY KCSDR HAROGIC SPECTRAN SPECTRAN_HTTP BADGESDR; do
  OFF="$OFF -DOPT_BUILD_${m}_SOURCE=OFF"
done
cmake -S $D/src -B $D/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$HOME/.local \
  -DOPT_BUILD_SOAPY_SOURCE=ON -DOPT_BUILD_M17_DECODER=OFF -DOPT_BUILD_DISCORD_PRESENCE=OFF $OFF
cmake --build $D/build -j4
cmake --install $D/build
```

The installed binary needs `LD_LIBRARY_PATH=$HOME/.local/lib` to find
`libsdrpp_core.so`.

### Configure the server

The default `~/.config/sdrpp/config.json` has no SoapySDR source instance.
Run SDR++ once to create the file, then add the instance under
`moduleInstances` and select it as the source:

```json
"moduleInstances": {
    "SoapySDR Source": { "module": "soapy_source", "enabled": true }
},
"source": "SoapySDR"
```

### Start and stop the server

```
cd ~ && LD_LIBRARY_PATH=$HOME/.local/lib \
  SOAPY_SDR_PLUGIN_PATH=$HOME/code/playground/hardware_hacking/diysdr/SoapyBirria/build \
  setsid nohup stdbuf -oL -eL ~/.local/bin/sdrpp --server --addr 0.0.0.0 --port 5259 \
  >$HOME/.cache/sdrpp-server.log 2>&1 </dev/null & disown
```

`setsid` and `disown` keep the server running after the shell that started
it exits. `stdbuf` keeps the log current. Stop the server with:

```
pkill -x sdrpp
```

### Connect from the Android app

1. Choose the source "SDR++ Server", enter the desktop's address and port
   5259, and tap Connect.
2. In the server's source menu, pick the birria device and a sample rate.
   The sample rate sets the waterfall span. Start at 5 MSps. 10 or 20 MSps
   work if the phone's Wi-Fi link keeps up. 40 MSps needs a wired client.
3. In the server settings, set the sample type to int8 and turn compression
   on. Without them, the stream needs two to four times the bandwidth.
4. Enter frequencies in Hz. For example, Wi-Fi channel 6 is `2437000000`.
   In the Frequency Manager, the Bandwidth field is the demodulator (VFO)
   width, not the waterfall span. Leave it at its default and use mode RAW,
   because SDR++ cannot demodulate Wi-Fi.

Good test frequencies are 2437 MHz (channel 6), 5220 MHz and 6135 MHz, where
access points were received during the library's qualification. Stop
playback before you change bands.

## Tests

`test_stream.cpp` captures through the plugin and reports the averaged
spectrum peak, to check decimation and the frequency axis. CMake does not
build it:

```
g++ -O2 -o build/test_stream test_stream.cpp -lSoapySDR
SOAPY_SDR_PLUGIN_PATH=$PWD/build build/test_stream <freq_hz> <rate> <seconds> [out.cf32]
```

To reproduce an application's problem, replay its call sequence against the
plugin: gain mode, gains, frequency, sample rate, then the stream. That is
how the SDR++ gain crash was found.
