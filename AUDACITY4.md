# Experimental Audacity 4 packaging

These changes were prepared by an LLM at the fork
owner's request. This is an unofficial packaging experiment, not an upstream
or Flathub-approved release.

## What changed

- Migrate the manifest from Audacity 3 to the checksum-pinned official Audacity
  4.0.0 offline source archive and KDE SDK/runtime 6.10 for its Qt interface.
- Point `EXTDEPS_CACHE` at the archive's bundled dependency sources. Prefer SDK
  libraries where supported; rebuild wxBase, PortAudio, LAME, pugixml and utfcpp
  from bundled sources. Compilation runs without network access.
- Build `patchelf` as a temporary dependency-packaging tool, removed at cleanup.
- Apply three patches: use SDK zlib/Expat/PCRE2 for wxBase to avoid its bundled
  zlib's GCC 15 failure; add the LinuxAudio VST3 extension search path; correct
  stale AppStream release, desktop-launcher and screenshot metadata.
- Retain LV2 discovery through the launcher wrapper and LinuxAudio 25.08
  extensions. Retain X11 for plugin compatibility and disable in-app updates.
- Stop applying obsolete Audacity 3 patches. Their files remain in the repo to
  keep this change small.

Upstream had already introduced the newer offline dependency approach. This
change uses it; it does not patch the beta dependency downloader reported in
[audacity#11157](https://github.com/audacity/audacity/issues/11157).
Related: [Flathub #202](https://github.com/flathub/org.audacityteam.Audacity/issues/202),
[failed version-bump PR #201](https://github.com/flathub/org.audacityteam.Audacity/pull/201),
and [upstream discussion #12139](https://github.com/audacity/audacity/discussions/12139).

The source archive's `OFFLINE_BUILD.md` identifies snapshot
`56bca1430b47cf33b815f578837284b0f5bf7325`, distinct from the release tag.
The manifest checksum pins the archive actually tested.

## Build and run without installing

Requires Flatpak Builder and the matching KDE 6.10 SDK/runtime. From this
checkout:

```sh
flatpak-builder --user --disable-rofiles-fuse --force-clean build org.audacityteam.Audacity.yaml
flatpak build --runtime --with-appdir --share=ipc --share=network \
  --socket=x11 --socket=pulseaudio --device=dri --filesystem=host \
  --filesystem=xdg-run/pipewire-0:ro --env=ALSA_CONFIG_PATH= build audacity
```

These commands do not install or replace Audacity. Downloads and build output
remain in `.flatpak-builder/` and `build/`. The app uses its normal settings
locations unless you provide separate XDG directories. Source downloads need
network access; compilation does not. After sources are cached, add
`--disable-download` to Builder to test an offline rebuild.

## Validation and limits

Local tests on Fedora 44 x86_64 with KDE SDK 6.10 completed source compilation
and installation, followed by incremental compilation/installation of the
final patches. Startup, WAV import with session database integrity checking,
the plugin-registration self-test, and loading MDA Ambience (LV2) and ZamComp
(VST3) passed. This does not establish that all plugins work.

AppStream validation and local manifest/build-directory lint passed with
Audacity's published Flathub exceptions. Lint recommends KDE 6.11. The local
catalog needed AppStream's `--no-partial-urls` option when preparing mirrored
media URLs; the linter environment was not Flathub's production Builder image.

Still required: clean builds of the final manifest on x86_64 and aarch64,
interactive playback/recording/export, AUP4 save/reopen, and native plugin
windows. Maintainers must also decide whether Audacity 4 belongs on stable,
beta, or a separate package; Audacity 3 workflows are not all supported in 4.

## Optional build without audio.com integration

Audacity provides a build-time option to disable audio.com cloud integration.
Add this entry under the audacity module's `config-opts` in the manifest:

```yaml
    config-opts:
      - -DAU_BUILD_CLOUD_AUDIOCOM=OFF
```

Rebuild the app for it to take effect; this is not a runtime environment
variable. The source uses this option to disable the cloud service and hide
related account, cloud project/audio, and export-preference UI. Some File menu
entries appear to be unconditional in this source snapshot, so the flag alone
may leave cloud references visible. It does not disable every network feature.
The default manifest retains cloud support.

Local testing of this optional configuration is in progress; user confirmation
of the resulting interface is still pending.
