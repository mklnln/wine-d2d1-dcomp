# Fixing black-window JUCE 8 / VSTGUI plugin GUIs under Wine on CachyOS / Arch

JUCE 8 switched its Windows renderer to **Direct2D**, and stock Wine / DXVK / vkd3d
don't implement Direct2D feature level 1.3. The result: a JUCE 8 plugin's GUI renders
as a **black (or blank) window** even though audio, MIDI, and automation work fine.
giang17's Wine fork (`d2d1-dcomp-11.0` branch) adds the missing Direct2D +
DirectComposition support and fixes it. patrickl ships this as a Fedora COPR; this is
the Arch/CachyOS equivalent.

**Rule of thumb:** if a plugin's window is black/blank under stock Wine but the audio
works, it's almost certainly this JUCE 8 / Direct2D issue.

Important: **not every JUCE 8 plugin breaks.** Only those that actually use the new
Direct2D/DComp render path *without* a Wine fallback. Some vendors already ship a
"detect Wine → fall back to the old software renderer" fix, so their JUCE 8 plugins
work on stock Wine with no patch needed.

> **Note on Pianoteq specifically:** Modartt ships a **native Linux build** of Pianoteq
> (standalone + native Linux VST3/LV2). If you just want Pianoteq, **use the native
> version** — it's faster, more stable, and needs no Wine, yabridge, or this patch at all.
> The Wine route below is only worth it for JUCE 8 plugins that have **no** native Linux
> build (e.g. Serum 2). Pianoteq is used here mainly as the confirmed test case.

Known affected (patch needed):
- **Pianoteq 9** — confirmed working here with this patched Wine (VST3, bridged into Bitwig).
  (But see the note above — the native Linux build is the better choice for Pianoteq itself.)
- **Serum 2** — reported broken on stock Wine (needs `d2d1.dll`; black boxes / crashes).
  See yabridge issue #413.

Known *fine* on stock Wine (vendor added a Wine fallback — no patch needed):
- **Minimal Audio Squash**, **Apulsoft apUnmask** — JUCE 8 but render correctly.

giang17's `PATCHES.md` also lists these as **target plugins to test** with the patch
(author's stated targets, not independently confirmed here): Serum 2, Korg Trinity /
Prophecy / Triton, Pianoteq 9, Garritan CFX, EZkeys 2.

If your plugin isn't listed, use the rule of thumb above: black window + working audio
under stock Wine ⇒ almost certainly needs this patch.

## The exact error / symptom we saw

The plugin GUI is a black/blank window, and Wine's console shows DirectComposition
failing — the plugin can't create its D2D/DComp device:

```
DCompositionCreateDevice failed: Not implemented. (0x80004001)
```

`0x80004001` is `E_NOTIMPL` ("not implemented") — stock Wine simply doesn't implement
`dcomp.dll`'s `DCompositionCreateDevice`, so the JUCE 8 Direct2D/DirectComposition
render path has nothing to draw into. The patched Wine implements it, and the GUI
renders normally.

## 1. Install the patched Wine (isolated, doesn't touch system Wine)

> **Not on the AUR.** The AUR froze all new account registrations in June 2026 after a
> [malware wave](https://www.theregister.com/security/2026/06/15/arch-linux-locks-down-aur-signups-amid-wave-of-malicious-commits/5255511),
> so this isn't published there — install straight from this repo instead (below). The
> `PKGBUILD`s here are AUR-ready if anyone wants to pick that up.

Clone the repo, then pick one option. Both install to `/opt/wine-d2d1`; your system
wine stays the default.

```sh
git clone https://github.com/mklnln/wine-d2d1-dcomp.git
cd wine-d2d1-dcomp
```

**Option A — prebuilt binary (fast, no compiling).** Downloads the already-compiled
build from this repo's GitHub release and drops it in place — seconds, not ~30-60 min.
makepkg verifies the download against the bundled sha256 checksum.

```sh
makepkg -p PKGBUILD-bin -si        # uses PKGBUILD-bin (the prebuilt one)
```

**Option B — build from source (slow, but fresh for your system).** Compiles giang17's
branch on your machine (~30-60 min); no prebuilt download.

```sh
makepkg -si                        # uses PKGBUILD (the source one)
```

Same software either way — the only difference is whether *you* compile it or install a
copy someone already compiled. See the two PKGBUILDs in this repo.

> **Non-Arch distros (Ubuntu/Debian/Fedora/etc.):** the patch and everything from Step 2
> onward are distro-agnostic and work anywhere Wine + yabridge do. But the packaging here
> is Arch-only: the **prebuilt binary won't run elsewhere** (it's compiled against Arch's
> glibc/libraries), and `makepkg`/`PKGBUILD` are Arch tooling. On other distros you'd
> compile giang17's `d2d1-dcomp-11.0` branch from source against **your own system's
> libraries** — that's a normal Wine-from-source build, but the specific build deps and
> steps are **out of scope for this repo**. Everything after you have a working
> `/opt/wine-d2d1` applies the same regardless of distro.

## 2. Make a dedicated prefix and install the plugin

```sh
export WINEPREFIX=~/.wine-<plugin> WINEARCH=win64
/opt/wine-d2d1/bin/wine wineboot -u
# recommended: real fonts so JUCE text isn't blank
WINE=/opt/wine-d2d1/bin/wine winetricks -q corefonts
# run your installer:
env WINEPREFIX=~/.wine-<plugin> /opt/wine-d2d1/bin/wine /path/to/setup.exe
```

## 3. Bridge into a native Linux DAW with yabridge — using the patched Wine only for this prefix

yabridge auto-detects the *prefix* from the plugin's location, but uses ONE Wine
binary for all plugins. To run only this prefix on the patched Wine (and keep every
other plugin on your system Wine), use a `WINELOADER` dispatcher:

`~/.local/bin/wine-yabridge-dispatch` (chmod +x):
```sh
#!/bin/sh
p="${WINEPREFIX%/}"
case "$p" in
    */.wine-<plugin>) exec /opt/wine-d2d1/bin/wine "$@" ;;  # patched
    *)                exec /usr/bin/wine        "$@" ;;  # system
esac
```

`~/.config/environment.d/yabridge-wine.conf`:
```
WINELOADER=/home/<you>/.local/bin/wine-yabridge-dispatch
```

Then register + sync, and log out/in so your DAW inherits the env:
```sh
yabridgectl add "$WINEPREFIX/drive_c/Program Files/Common Files/VST3"
yabridgectl sync
```

To test without logging out:
`env WINELOADER=~/.local/bin/wine-yabridge-dispatch bitwig-studio`

## Will this stop being necessary? (upstream status)

Probably, eventually — this is a **stopgap**, not a forever thing. Mainline Wine is
actively working toward the same goal:

- **Wine-Staging 11.6** shipped a large (~65-patch) **DirectComposition** series from
  CodeWeavers, and later staging adds `DCompositionCreateDevice2`. That's the `dcomp`
  half of the problem — the exact call in the error above.
- **But it's not enough on its own yet.** Stock Wine/staging still implements only
  **Direct2D 1.2**, while JUCE 8 wants **1.3**. That missing D2D 1.3 path is what
  giang17's patch supplies, and it is **not** in mainline/staging as of this writing.
  So today you still need this patch (or giang17's branch on top of staging).
- Trajectory: once mainline Wine lands full Direct2D 1.3 + DirectComposition, plain
  Wine/Proton/staging should render these plugins natively and this package becomes
  unnecessary. Until then, this works now.

Note: **wine-valve** (Valve's Proton fork) does *not* include this fix — its `d2d1` is
the same incomplete built-in module every Wine has. Installing it won't help.

## Notes
- Built with new-WoW64 (`--enable-archs=i386,x86_64`) so no 32-bit dev libs needed.
- `d2d1-dcomp-11.0` is the maintainer's recommended, plugin-tested branch (the
  `-11.8` branch is a frozen snapshot for wine-tkg-dev users).
- Credit: patch author **giang17** (github.com/giang17/wine), COPR by **patrickl**.
