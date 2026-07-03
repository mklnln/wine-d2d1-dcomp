# Success report to send upstream

## Where to post
- **giang17 (patch author)** — open a GitHub Discussion/Issue on
  https://github.com/giang17/wine  (PATCHES.md actively asks for plugin-test feedback).
- **patrickl (COPR) / yabridge community** — yabridge Discord, #general
  (invite from the COPR/Fedora thread: https://discord.gg/pyNeweqadf).
- Optional: reply on the Fedora thread
  https://discussion.fedoraproject.org/t/patrickl-wine-11-8-vstgui-juce8/190394

## Draft message

> **Success report — Pianoteq 9 working on Arch/CachyOS via d2d1-dcomp-11.0**
>
> Built the `d2d1-dcomp-11.0` branch from source on CachyOS (Arch, rolling), Wine 11.0,
> new-WoW64 (`--enable-archs=i386,x86_64`, no lib32 deps), installed isolated to
> `/opt/wine-d2d1`. Compiled clean, no patch/build issues.
>
> Pianoteq 9 (VST3, 64-bit) now renders its full JUCE8 GUI correctly — no black window.
> Running it bridged into **Bitwig Studio** via **yabridge 5.1.1**. Because yabridge uses
> one Wine binary globally, I use a `WINELOADER` dispatcher script that routes only the
> Pianoteq prefix to the patched Wine and everything else to system wine-staging 9.21 —
> works perfectly, other plugins unaffected.
>
> I packaged it for Arch/CachyOS (source + prebuilt binary PKGBUILDs, plus a full
> write-up incl. the yabridge dispatcher) here, in case it's useful to point other Arch
> users at: https://github.com/mklnln/wine-d2d1-dcomp
> (Not on the AUR — new-account registration is frozen after the June 2026 malware wave —
> so it's install-from-repo for now.)
>
> Just wanted to confirm the 11.0 branch works great outside Fedora and thank you for the
> patches. Happy to test other plugins or provide build details.

## Notes
- Repo is public and credits you as the patch author throughout. Feel free to link it.
- If AUR registration reopens later, the PKGBUILDs are already AUR-ready.
