# R2D2 reSpeaker Flex XVF3800 handoff

Updated November 5, 2026 (America/Chicago).

## Current state: working

The user confirmed the playback fix compiled, uploaded, and is now working. Okay Nabu wakes R2D2, microphone audio reaches Home Assistant, the question is recognized, and the spoken answer plays. The test was: “Okay Nabu … What time is it?” Earlier unavailable entities and the failed voice-setup connectivity test are historical symptoms, not the current baseline.

- Hardware: Seeed reSpeaker Flex XVF3800 Circular with XIAO ESP32-S3.
- Home Assistant device: R2D2, Master Bedroom.
- Assistant: Home Assistant Cloud; conversation agent: Home Assistant.
- Wake word: Okay Nabu. Second wake word: No wake word.
- Last inspected: microphone mute off, wake sound on, sensitivity Slightly sensitive.
- Preserve eight WS2812 LEDs on GPIO4 and R2D2 audio effects.

## Repository and toolchain

- Repository: `C:\Dev\R2D2-reSpeaker-Flex-XVF3800`.
- Origin (user-owned fork; fetch/push): https://github.com/crnewman/R2D2-reSpeaker-Flex-XVF3800.git
- Upstream (original project; fetch/reference): https://github.com/TitoTB/R2D2-reSpeaker-Flex-XVF3800.git
- Working branch: `r2d2-esphome-2026-10-4`.
- Tracking branch: `origin/r2d2-esphome-2026-10-4`.
- GitHub branch: https://github.com/crnewman/R2D2-reSpeaker-Flex-XVF3800/tree/r2d2-esphome-2026-10-4
- The user confirmed the working fix and handoff were committed, then supplied successful push output confirming this branch was created on the fork and upstream tracking was set. Future pushes from this branch can use `git push`. No pull request or merge was reported.
- Historical pre-fix commit: `267971e` (Compile R2D2 with ESPHome 2026.10.4 integration). The latest commit hash was not supplied; inspect current HEAD rather than treating this historical hash as current.
- Active configuration: `R2D2-2026.10.4.yaml`. Original `R2D2.yaml` preserved.
- Windows 11; `.venv`; prior setup reports Python 3.14.2. ESPHome CLI/runtime 2026.10.2.
- Filename/tag 2026.10.4 identifies the external integration revision, not the CLI version. Project metadata still says 2026.6.0.
- Build root: `C:\Dev\esphb`; device build: `C:\Dev\esphb\r2d2\build`.
- Network target: `r2d2.local`. USB port last verified: COM7; recheck when needed.
- Upstream checkout: `C:\Dev\Respeaker-XVF3800-ESPHome-integration`, previously tag 2026.10.4 detached HEAD. Components are under `esphome\components`.

## Speech-playback fix: applied and user-tested

Symptom: R2D2 wake/command sounds played, but it did not speak the answer. A direct announcement, “R2D2 audio connection test,” played successfully.

HA Cloud debug showed correct recognition of “What time is it?”, successful HassGetCurrentTime, response “7:32 PM,” and successful TTS generation using tts.home_assistant_cloud / AndrewNeural.

The YAML's voice_assistant.on_tts_end called play_sound with priority: true for command_complete_sound. That script stops the announcement player before playing its sound. Installed ESPHome source confirms on_tts_end fires after submitting the speech URL, not after spoken playback finishes. The completion sound could therefore replace the spoken response.

The fix queues the completion sound behind speech while preserving the TTS URI event:

```yaml
  on_tts_end:
    - script.execute:
        id: send_tts_uri_event
        tts_uri: !lambda 'return x;'
    # TTS_END delivers the response URL; speech has not finished playing.
    # Queue the R2D2 completion sound instead of stopping the spoken answer.
    - media_player.enqueue:
        id: external_media_player
        media_url: "audio-file://command_complete_sound"
        announcement: true
    - light.turn_off: small_led
    - light.turn_off: big_led
```

The assistant reread the local YAML and verified this block is present. The user ran the corrected helper, confirmed compilation/upload, and reported working voice responses. No XMOS firmware was flashed. New binary size/hash and post-fix Git commit were not independently recorded; do not reuse the old factory-image hash as the current build identity.

## Configuration and safeguards

- respeaker_xvf3800 and aic3104 pinned to formatBCE/Respeaker-XVF3800-ESPHome-integration tag 2026.10.4.
- decimator pinned to formatBCE/Respeaker-Lite-ESPHome-integration tag 2026.10.4; obsolete custom I2S fork removed.
- Raw microphone i2s_mics_raw: 48 kHz, 32-bit stereo, secondary I2S, DIN GPIO43. Decimator i2s_mics selects channels [0, 1] for 16 kHz processing. Wake-word input uses channel 1; voice-assistant input channel 0.
- I2C SDA GPIO5, SCL GPIO6, 100 kHz; scan enabled. XMOS address 0x2C; AIC3104 driver default 0x18.
- XMOS version last observed: 1.0.7.
- NEVER automatically flash XMOS firmware. Flex Circular differs from generic/compact hardware. Keep the respeaker_xvf3800 firmware auto-update block absent.
- API encryption references !secret api_key; fallback AP references !secret fallback_ap_password. Never print or commit secret values. secrets.yaml is ignored.
- IPv6 enabled, Wi-Fi power saving disabled. Logging INFO, with I2S WARN.
- Preserve the original YAML and custom LED/audio behavior.
- Do not indiscriminately stage 3D and Docs/, .venv/, .esphome/, logs, generated binaries, or backup files.

## Network and diagnostic findings

- Encrypted API handshakes and COM7 runtime logging succeeded earlier in the session.
- HA Local network URL was Automatic, using a numeric LAN IPv4 address on port 8123. No DNS lookup was needed for that configured endpoint. No DNS or HA URL settings were changed.
- Direct announcement playback and the successful voice-response test confirm the audio path works. Do not change DNS based solely on the earlier wizard error.
- The Sendspin proxy setup banner is separate from the working voice functionality; it was not needed for this test.
- Earlier HA logs showed an AssertionError at components/esphome/update.py:144 while creating a dashboard firmware-update entity. Its relationship to earlier availability problems was not established. Do not treat it as a current voice failure without new evidence.
- The Assist connector found no exposed R2D2 entities despite the working browser UI. Connector exposure does not prove whether a device is online.
- Edge browser tools successfully accessed the authenticated HA UI after an earlier native computer-use attempt failed URL identification.

## Build and upload

From the repository in PowerShell, run each command only after the preceding command succeeds:

```powershell
.\.venv\Scripts\python.exe -m esphome compile .\R2D2-2026.10.4.yaml
.\.venv\Scripts\python.exe -m esphome upload .\R2D2-2026.10.4.yaml --device r2d2.local
.\.venv\Scripts\python.exe -m esphome logs .\R2D2-2026.10.4.yaml --device r2d2.local
```

Logs run continuously. Ctrl+C stops the viewer without stopping R2D2. Use --device COM7 for serial logging after checking the port.

The one-off Apply-R2D2-Speech-Fix.ps1 helper initially hung before compilation because of a pathological nested regex in its XMOS safety check. It was corrected to scan line by line (tested in 21 ms), and the user successfully ran the corrected script. Do not reuse the old regex. A timestamped R2D2-2026.10.4.yaml.before-speech-fix-*.bak exists locally; do not stage it accidentally.

Later chat permissions allowed reading the repository but blocked direct writes/builds/uploads outside the chat workspace. The user executed the helper locally. Check permissions next time rather than assuming desktop access grants unrestricted execution.

## Git and next steps

This handoff is intended to be committed with the configuration. It contains no credential values. Private server addresses/account-specific browser URLs are omitted. It includes local project paths and room/device names; remove these if the owner prefers not to publish them.

1. Inspect `git status --short`, `git --no-pager log -1 --oneline`, and `git remote -v` for changes since the confirmed push. The working fix and earlier handoff were committed and pushed; this repository/branch documentation update needs its own commit and push after installation.
2. Preserve the working baseline. Optional follow-up tests: normal reboot, repeated questions, and a longer spoken response. These additional tests have not been recorded.
3. Continue enclosure/assembly or feature work based on the user's next priority. Do not reflash or change settings merely to resume.
4. For a new voice failure, inspect the latest HA Cloud assistant debug run, distinguishing microphone/STT, intent, TTS generation, and device playback.

Suggested explicit staging (review the staged diff before committing):

```powershell
git add -- R2D2_HANDOFF.md
git --no-pager diff --cached
git commit -m "Document GitHub fork and tracking branch"
git push
```

## Resume prompt

Read R2D2_HANDOFF.md and inspect current Git status. R2D2 is a working Seeed reSpeaker Flex XVF3800 Circular Home Assistant voice satellite with Okay Nabu wake word. Its speech fix queues the R2D2 completion sound instead of interrupting TTS. Preserve the working configuration, original YAML, and LED/audio customization. Never expose secrets, automatically flash XMOS, or indiscriminately stage reference folders/backups. Continue from the user's next requested improvement rather than restarting the resolved connectivity diagnosis.

## References

- https://github.com/crnewman/R2D2-reSpeaker-Flex-XVF3800/tree/r2d2-esphome-2026-10-4
- https://aguacatec.es/disenos-3d/r2d2-respeaker-flex-xvf3800/
- https://aguacatec.es/academia/r2d2-respeaker-flex-xvf3800/
- https://github.com/TitoTB/R2D2-reSpeaker-Flex-XVF3800
- https://github.com/formatBCE/Respeaker-XVF3800-ESPHome-integration/releases/tag/2026.10.4
