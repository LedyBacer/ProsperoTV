# FW 3.00 AGC-first startup hardware test

This PR changes only the released OpenGL UI assembly path. It is an
**experimental** firmware compatibility fix, pending a FW 3.00 and current-FW
hardware check. It does **not** alter VideoDec2 codec, stream parsing, or VRR.

## Why

On FW 3.00, ProsperoLight testing isolated a startup order problem:
`sceVideodec2QueryComputeMemoryInfo` before AGC initialization could cause
`sceAgcGetRegisterDefaults` to be NULL and leave a black screen. Initializing
AGC first restored the menu. ProsperoTV's `iptv_native_backend_warm` similarly
used to run that compute query immediately on entry, ahead of OpenGL.

This build:
1. Preloads the VideoDec2 module **before** EGL (preserves loader constraints).
2. Initializes the existing OpenGL runtime's AGC via a temporary EGL Display
   context, then closes the context.
3. Warms compute, H.264/HEVC/VP9, and AAC **after** AGC, still **before**
   filesystem elevation.
4. Opens the normal menu display and player later, as before.

The temporary EGL context is deliberate: it avoids calling
`sceAgcInit` directly with the wrong API signature or bypassing the
OpenGL runtime's internal initialized-state bookkeeping.

## How to test

The CI workflow for this PR's branch builds the released UI with
`TV_TEST_TITLE=PPSA88003` and `TV_DEBUG_TRACE=1`. Use the **app** artifact,
`ProsperoTV-app-PR<id>-<sha>`, **not** the root legacy-app artifact.
After extracting the GitHub artifact, install the enclosed
`PPSA88003.zip` as a separate title. It must **not** replace the released
`PPSA99003` installation or share its config.

On FW 3.00, first start the app and confirm the menu paints and responds.
Then try an H.264 TV channel, HEVC if available, and return to the menu.
Repeat on a newer FW. Inspect the build label, the output/decoder startup
receipts, and the forced debug log. Preserve any fresh crash report and note
the exact FW and enabler version. If it crashes before logs initialize,
capture kernel/debug output: the bootstrap occurs before per-user storage.

Look for `[TV][FWCompat]` entries:
- video module preload rc
- early AGC via EGL opened and EGL error
- decoder warm-up after AGC compute/H.264/HEVC/VP9 return codes

Do not merge before receiving hardware confirmation of both startup and
playback. Initializing a throwaway EGL context this early also changes
threading/display lifecycle timing, which must be verified on hardware.
