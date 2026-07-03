# Reddit post draft

**Suggested subreddits:** r/linuxaudio (best fit), r/cachyos, maybe r/modartt / r/synthesizers.
Fill in the three `<link>` placeholders before posting.

---

**Title:** Got JUCE 8 plugins (Serum 2, etc.) working on Arch/CachyOS Wine — patched Wine + prebuilt package so you don't have to compile

---

If you've updated a plugin recently and its window is now just **black** under Wine/yabridge
even though the audio works fine — this is for you. There's a fix, and I've packaged it so
you don't have to compile Wine yourself.

**First, the important caveat:** if the plugin you care about has a **native Linux build,
use that instead.** Pianoteq, for example, ships a proper native Linux version — don't
bridge it through Wine. This whole guide is really for the plugins that are **Windows-only**
and shipped a JUCE 8 update that broke their GUI (Serum 2 being the headline one). I used
Pianoteq 9 as my known-good test case, but for Pianoteq itself the native build is strictly
better.

**I didn't write any of the actual fix** — all credit to [giang17](https://github.com/giang17/wine),
who wrote the Wine patches, and patrickl, who packages them for Fedora. I just built it for
Arch and wrote down the steps. I'm not a Wine dev; if I can do this, you can too.

## The problem

JUCE 8 changed its Windows renderer to **Direct2D / DirectComposition**. Stock Wine (and
DXVK/vkd3d) don't fully implement that path — so a JUCE 8 plugin that uses it renders a
black or blank window. Audio, MIDI, and automation all work; you just can't see the UI.

If you launch the plugin from a terminal, the tell-tale sign is:

```
DCompositionCreateDevice failed: Not implemented. (0x80004001)
```

That `0x80004001` is `E_NOTIMPL` — Wine literally hasn't implemented the DirectComposition
call the plugin needs, so there's nothing for it to draw into.

**Not every JUCE 8 plugin is affected.** Some vendors (Minimal Audio, Apulsoft) added a
"detect Wine → use the old software renderer" fallback, so their plugins work on plain Wine.
The ones that break are those using the new render path with no fallback. Rule of thumb:
**black window + working audio = this bug.**

## The fix

[giang17's Wine fork](https://github.com/giang17/wine) has a branch (`d2d1-dcomp-11.0`) that
implements the missing Direct2D + DirectComposition bits. You build that, install it
**isolated** so it doesn't touch your system Wine, and point only the affected plugin at it.

I packaged it two ways for Arch/CachyOS:

- **`wine-d2d1-dcomp-bin`** — prebuilt. I already compiled it; you just install it. Seconds,
  no 40-minute build. *(recommended)*
- **`wine-d2d1-dcomp-git`** — builds from source on your machine if you'd rather not trust a
  stranger's binary (fair). ~30–60 min.

Either way it installs to `/opt/wine-d2d1` and **leaves your system Wine as the default.**
Nothing else on your system changes.

**Not on the AUR** — the AUR froze new registrations in June 2026 after the malware wave,
so I can't publish there right now. Install straight from the GitHub repo instead:

```sh
git clone https://github.com/mklnln/wine-d2d1-dcomp.git
cd wine-d2d1-dcomp

# prebuilt (fast — downloads the release binary, checksum-verified):
makepkg -p PKGBUILD-bin -si

# or build it yourself from source (~30–60 min):
makepkg -si
```

## Set up a dedicated prefix for the plugin

Keep this in its own Wine prefix so it's isolated from your main setup. Install your plugin
into it with the patched Wine:

```sh
export WINEPREFIX=~/.wine-<plugin> WINEARCH=win64
/opt/wine-d2d1/bin/wine wineboot -u

# real fonts, or JUCE text renders blank:
WINE=/opt/wine-d2d1/bin/wine winetricks -q corefonts

# run your installer:
env WINEPREFIX=~/.wine-<plugin> /opt/wine-d2d1/bin/wine /path/to/installer.exe
```

## The tricky part: making yabridge use the patched Wine for *only* this plugin

Here's the catch that took me a while. **yabridge uses one Wine binary for every plugin.**
You can't tell it "use patched Wine for Serum but system Wine for everything else" through
any setting. It auto-detects the *prefix* from where the plugin lives, but the *binary* is
global.

The trick is a tiny **dispatcher script** you set as `WINELOADER`. yabridge sets `$WINEPREFIX`
per-plugin, so the script routes based on that: patched Wine for your plugin's prefix, system
Wine for all the others.

`~/.local/bin/wine-yabridge-dispatch` (make it executable — `chmod +x`):

```sh
#!/bin/sh
p="${WINEPREFIX%/}"
case "$p" in
    */.wine-<plugin>) exec /opt/wine-d2d1/bin/wine "$@" ;;  # patched
    *)                exec /usr/bin/wine        "$@" ;;  # system default
esac
```

Then tell your session to use it, via `~/.config/environment.d/yabridge-wine.conf`:

```
WINELOADER=/home/<you>/.local/bin/wine-yabridge-dispatch
```

Register the plugin and sync:

```sh
yabridgectl add "$WINEPREFIX/drive_c/Program Files/Common Files/VST3"
yabridgectl sync
```

`environment.d` is applied at login, so **log out and back in** and your DAW (launched from
the menu) will inherit it. To test *without* logging out, just launch the DAW from a terminal
with the var set:

```sh
env WINELOADER=~/.local/bin/wine-yabridge-dispatch bitwig-studio
```

Open the plugin — the GUI should now render properly, and every other plugin keeps using
your normal Wine untouched.

## Will this become unnecessary? (yes, eventually)

This is a **stopgap**. Mainline Wine is working toward the same thing: Wine-Staging 11.6
shipped a big DirectComposition patch series (the `dcomp` half of the error above), and it's
progressing. **But it's not enough yet** — stock Wine/staging still only implements Direct2D
**1.2**, and JUCE 8 wants **1.3**. That missing D2D 1.3 path is exactly what giang17's patch
supplies, and it isn't in mainline/staging yet. So as of now you still need this.

Two things people ask:
- **Does wine-staging already fix it?** No — see above. The DComp work landed; the D2D 1.3
  part hasn't.
- **What about wine-valve / Proton?** Also no. Its `d2d1` is the same incomplete built-in
  module every Wine has; it doesn't include this patch.

Once mainline lands full Direct2D 1.3 + DirectComposition, plain Wine should handle these
plugins natively and this package can be retired. Until then, this works today.

## Credits / links

- **[giang17](https://github.com/giang17/wine)** — wrote the actual Direct2D/DirectComposition
  Wine patches. All the hard work is theirs.
- **patrickl** — maintains the [Fedora COPR](https://copr.fedorainfracloud.org/coprs/patrickl/wine-11.8-vstgui-juce8/)
  version; this Arch packaging is just the same idea ported over.
- **[yabridge](https://github.com/robbert-vdh/yabridge)** by robbert-vdh — the bridge itself.
- **Repo + full write-up:** https://github.com/mklnln/wine-d2d1-dcomp
- **Prebuilt binary:** https://github.com/mklnln/wine-d2d1-dcomp/releases/tag/v11.0

Happy to answer questions — though for anything about the patches themselves, giang17's repo
is the place.
