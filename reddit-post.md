# Reddit post draft (short "signpost" version)

**Suggested subreddits:** r/linuxaudio (best fit), r/cachyos, maybe r/modartt / r/synthesizers.

---

**Title:** Fix for black-window JUCE 8 plugin GUIs (Serum 2, etc.) under Wine on Arch/CachyOS

---

If you've updated a plugin recently and its GUI is now just a **black window** under
Wine/yabridge — even though audio, MIDI, and automation still work — that's the JUCE 8
Direct2D/DirectComposition issue. Stock Wine doesn't implement the render path JUCE 8
switched to, so there's nothing to draw into. The tell-tale sign, if you launch from a
terminal:

    DCompositionCreateDevice failed: Not implemented. (0x80004001)

**First, a caveat:** if the plugin you want has a **native Linux build, just use that**
(Pianoteq, for example) — this is only for **Windows-only** JUCE 8 plugins like Serum 2.

There *is* a fix — [giang17](https://github.com/giang17/wine) wrote Wine patches that
implement the missing bits — but building patched Wine and wiring it into yabridge for
just one plugin (without disturbing your other plugins) is fiddly. So I packaged it for
Arch/CachyOS and wrote up the whole thing, including a **prebuilt binary** so you can skip
the ~40-min compile, and the yabridge dispatcher trick that routes *only* the patched
plugin to the patched Wine:

**Full guide + prebuilt binary:** https://github.com/mklnln/wine-d2d1-dcomp

Not on the AUR (new registrations are frozen after the June 2026 malware wave), so it's
install-straight-from-the-repo for now — instructions are in the README.

All credit for the actual fix goes to giang17; I just packaged it and wrote it down. Happy
to answer questions here, but anything about the patches themselves is best directed at
their repo.
