# Lenovo LOQ 16IRH8 speaker EQ for Linux (PipeWire)

A ready-made PipeWire filter that makes the built-in speakers of the **Lenovo LOQ 16IRH8**
sound noticeably better on Linux: less tinny, louder, and with some perceived bass.

Out of the box on Linux, the speakers get the raw signal with no processing. On Windows,
Lenovo's audio software applies EQ and loudness processing, which is why they sound better
there. This config does the same job using only PipeWire and three small open-source
LADSPA plugins.

It shows up as a new output device called **"Speakers (DSP multiband)"**. Pick it in your
sound settings and you're done. To compare with the original sound, switch back to the
normal built-in output.

## What it does

The EQ is not guesswork. It was fitted to real measurements: a sine sweep was played
through the speakers and recorded with a phone at five positions around the listening
spot, and the results were averaged. The phone mic was checked against a MacBook Pro's
speakers to confirm it records bass correctly.

**What the measurement found** (relative to the 1.5–4 kHz level):

| Range | Raw speakers |
|---|---|
| Below ~600 Hz | falls off steeply (−9 dB at 500 Hz, −20 dB at 300 Hz) |
| 800–1300 Hz | a broad **+7 to +8 dB hump**, which causes the tinny, "honky" sound |
| Above 5 kHz | about −3 to −6 dB (dull treble) |

**The processing chain**, for each channel:

```
input ─► correction EQ ─► 3-band split (500 Hz / 3 kHz) ─► compressor per band ─┐
                                                                                ├─► mix ─► limiter ─► speakers
input (L+R) ─► virtual bass (harmonics 100–650 Hz) ─► its own compressor ───────┘
```

- **Correction EQ.** A broad −12 dB cut at 1070 Hz removes the hump, plus a small −2 dB cut
  at 3 kHz. A high-pass at 150 Hz keeps deep bass the speakers can't play out of them,
  which reduces distortion and frees up loudness.
- **Three-band compression.** The bass, mids and treble are compressed separately, so a
  loud kick drum doesn't turn the vocals down. This is what makes the speakers sound
  louder, much like the loudness processing laptop audio software uses on Windows.
- **Virtual bass.** These speakers can't reproduce real bass. Instead, the deep bass is
  converted into its higher harmonics (60 Hz becomes 120 Hz, 240 Hz, …), which the speakers
  *can* play, and your brain fills in the missing low note. This is the same trick
  commercial "bass enhancers" use.
- **Limiter.** Stops the louder signal from clipping, with a ceiling of −1 dBFS.

Measured result: the response between 650 Hz and 12 kHz goes from about ±3.4 dB to
±2.5 dB, and the 1 kHz hump drops from +8 dB to about +2 dB.

Be realistic about bass. Below roughly 400 Hz these speakers produce 25–35 dB less than a
MacBook Pro's, and no software can change that. The virtual bass makes bass lines audible;
it can't make the speakers thump. For real bass, use headphones or an external speaker.

## Does it work on my laptop?

- **Tested on:** Lenovo LOQ 16IRH8 (Intel, Realtek ALC257 codec), Fedora Linux 44/45,
  PipeWire 1.6.
- **Other LOQ models** with the same chassis and speakers will probably benefit, but the EQ
  was measured on one unit.
- **Other laptops:** the general approach applies, but the EQ values are specific to these
  speakers. Using them on different speakers will likely make things worse.

To check which codec you have:

```sh
cat /proc/asound/card*/codec#* | grep Codec
```

## Requirements

- **PipeWire** with WirePlumber. This is the default on Fedora, Ubuntu 22.10+, Arch and
  most current distros.
- **The SWH LADSPA plugins.** The config uses three of them: `sc4_1882` (stereo
  compressor), `sc4m_1916` (mono compressor) and `fast_lookahead_limiter_1913` (limiter).

## Installation

### 1. Install the plugins (recommended: from your distro)

```sh
# Fedora
sudo dnf install ladspa-swh-plugins

# Debian / Ubuntu
sudo apt install swh-plugins

# Arch
sudo pacman -S swh-plugins
```

Using the distro package is the reliable route. PipeWire finds the plugins automatically,
and they get security and compatibility updates.

#### Alternative: the bundled plugin files

The `plugins/` folder contains the three plugin files, copied from Fedora's
`ladspa-swh-plugins-0.4.17-21.fc45` (x86_64). They are only for systems where you can't
install the package. To use them:

1. Copy them somewhere permanent, for example `~/.local/lib/ladspa/`.
2. In the config file, replace each `plugin = <name>` with the full path to the file,
   for example `plugin = "/home/you/.local/lib/ladspa/sc4_1882.so"`.

These are x86_64 Linux builds that need only the standard C and maths libraries. This
route hasn't been tested as thoroughly as the distro package.

### 2. Check your speaker output name

The config sends its output to a specific device. Find yours:

```sh
pactl list short sinks
```

Look for the one ending in `analog-stereo` on the Intel audio controller. On the
LOQ 16IRH8 it is:

```
alsa_output.pci-0000_00_1f.3.analog-stereo
```

If yours is different, edit this line near the bottom of
`sink-laptop-speakers-dsp.conf`:

```
target.object  = "alsa_output.pci-0000_00_1f.3.analog-stereo"
```

### 3. Install the config and restart PipeWire

```sh
mkdir -p ~/.config/pipewire/pipewire.conf.d
cp sink-laptop-speakers-dsp.conf ~/.config/pipewire/pipewire.conf.d/
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### 4. Select it

Open your sound settings and choose **"Speakers (DSP multiband)"** as the output. Your
choice is remembered across reboots. From the command line:

```sh
wpctl status            # find the ID next to "effect_input.speakers_dsp"
wpctl set-default <ID>
```

## Tuning

Everything is in `sink-laptop-speakers-dsp.conf`, and the header comments explain each
setting. Each channel has its own copy of the EQ, so **change the `_L` and `_R` lines
together**. After any edit, run:

```sh
systemctl --user restart pipewire pipewire-pulse wireplumber
```

| You hear | Change |
|---|---|
| Not enough bass | Raise `"Gain 4"` on the `out_L`/`out_R` lines (default 2.0; try 2.5) |
| Crackling or rattling on bass-heavy music | Lower `"Gain 4"`, or lower `comp_low`'s `Threshold level (dB)` (e.g. −24) |
| Not loud enough | Raise `Makeup gain (dB)` by the same amount on all three `comp_*` compressors |
| Sounds squashed or "pumping" | Raise the `comp_*` thresholds by a few dB |
| Dull or lacking sparkle | Raise `air_L`/`air_R` `Gain` to +2 or +3 |
| Lacks body | Raise `box_L`/`box_R` `Gain` to +2 or +3 |

Keep `Makeup gain` equal across the three compressors. If they differ, the EQ balance
shifts.

## Troubleshooting

**No sound at all, even though nothing is plugged into the headphone jack.**
The built-in output sometimes switches itself to the headphone port, which mutes the
speakers. Switch it back:

```sh
pactl set-sink-port alsa_output.pci-0000_00_1f.3.analog-stereo analog-output-speaker
```

or pick **Speakers** as the port in your sound settings.

**"Speakers (DSP multiband)" doesn't appear.**
Check the log for errors. A missing plugin is the usual cause:

```sh
journalctl --user -u pipewire -b | grep -iE "filter|ladspa|error"
```

**Wired headphones get the speaker EQ.**
On this laptop the speakers and the headphone jack are the same output device with two
ports. When using wired headphones, switch the output back to the normal built-in device.
Bluetooth is not affected.

## Uninstall

```sh
rm ~/.config/pipewire/pipewire.conf.d/sink-laptop-speakers-dsp.conf
systemctl --user restart pipewire pipewire-pulse wireplumber
```

## Licences

- The bundled plugins in `plugins/` are from Steve Harris's
  [SWH LADSPA plugins](https://github.com/swh/ladspa), licensed under the GNU GPL v2. See
  `plugins/COPYING`. The source code for these builds is available from upstream
  (version 0.4.17) and from Fedora's `ladspa-swh-plugins-0.4.17-21.fc45` source package.
