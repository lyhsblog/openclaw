---
summary: "On-device wake-word listening in the iOS and Android companion apps"
read_when:
  - Enabling or debugging Voice Wake on iPhone or Android
  - Comparing mobile wake-word behavior with macOS Voice Wake
  - Checking why a mobile node does not advertise voiceWake
title: "Voice wake (iOS and Android)"
---

Voice Wake on iPhone and Android is **in-app wake-word listening** while OpenClaw is visible. It is not Siri, Google Assistant, or an always-on system digital-assistant role. The Gateway still owns the shared wake-word list; each phone only stores a local enable toggle plus a cached copy of that list.

For the Gateway list, protocol, and routing rules, see [Voice wake](/nodes/voicewake). For the macOS menu-bar app, see [Voice wake (macOS)](/platforms/mac/voicewake).

## Requirements

- A paired [iOS](/platforms/ios) or [Android](/platforms/android) node with a reachable Gateway.
- Microphone access. iOS also needs Speech Recognition permission.
- Android needs on-device speech recognition (`SpeechRecognizer.isOnDeviceRecognitionAvailable`). If that contract is missing, the Voice Wake toggle stays unavailable.
- iOS Voice Wake does not run in Simulator.

Voice Wake is off by default. Chat dictation, voice notes, and [Talk mode](/nodes/talk) stay available without it.

## Quickstart

1. Pair the phone and confirm the Gateway connection in Settings.
2. Open Voice settings:
   - iOS: **Settings → Voice & Talk**
   - Android: **Settings → Voice**
3. Enable **Voice Wake** (iOS) or **Listen for wake words** (Android). Grant microphone (and on iOS, speech) permission if asked.
4. Keep the wake-word list short. Defaults after sanitizing an empty list are `openclaw` and `claude` on iOS, and `openclaw`, `claude`, and `computer` on Android. After Gateway sync, both apps use the Gateway list.
5. Leave OpenClaw in the foreground. Say a wake word, pause, then the command: `openclaw, what's on my calendar`.
6. Confirm the listener status shows **Listening**, then **Triggered**. The command is sent as a `voice.transcript` node event to the node's main session.

## Configuration

Wake words are one global Gateway list. Editing them on the phone calls `voicewake.set`; other clients receive `voicewake.changed`.

- At most 32 triggers; each trigger is trimmed and truncated to 64 UTF-16 code units.
- An empty saved list falls back to that platform's local defaults until the Gateway list arrives.
- Android can save wake words only while connected. Disconnected edits show **Connect to a Gateway to save wake words**.
- iOS writes the local cache immediately, then syncs to the Gateway after a short debounce.

While Voice Wake is enabled and available, the node advertises the `voiceWake` capability. Android also requires microphone permission and a wake-word list that has finished syncing for the current Gateway.

## Runtime behavior

Both apps listen with the platform speech recognizer, match a trigger, then forward only the command text.

| Surface    | iOS                                              | Android                                                                                                          |
| ---------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Recognizer | `SFSpeechRecognizer` plus an `AVAudioEngine` tap | On-device `SpeechRecognizer` (`EXTRA_PREFER_OFFLINE`)                                                            |
| Match      | Trigger plus a ~0.45s pause before the command   | Final transcript only; leading fillers such as "hey" / "嗯" are ignored                                          |
| Scripts    | Spoken English word timing                       | Word boundaries for spaced scripts; CJK and similar scripts match without spaces                                 |
| Forwarding | `voice.transcript` to the main session           | `voice.transcript` to the main session                                                                           |
| Capability | Advertised while the local toggle is on          | Advertised while enabled, on-device recognition is available, microphone is granted, and Gateway words are ready |

Android ignores partial transcripts. The on-device recognizer often stops mid-phrase, so only a final result is safe to dispatch.

## When listening pauses

Voice Wake yields whenever another OpenClaw surface owns the microphone or the app is no longer visible. Each owner clears only its own pause reason, so Talk ending cannot restart listening over an active voice note.

iOS pauses during backgrounding, Talk, push-to-talk, voice-note capture, and auxiliary audio.

Android pauses when the app is not in the foreground, and during camera capture, Chat dictation, voice-note capture, Talk / voice capture, message speech, spoken replies, and Gateway wake-word sync.

Status text is **Paused** while a reason is active, then the listener starts again when the last reason clears.

## Troubleshooting

- **Listener stays Off or unavailable:** Android needs on-device speech recognition for the device language. iOS needs Microphone and Speech Recognition in the iOS Settings app under OpenClaw. Simulator is unsupported.
- **Listening never appears:** keep the app foregrounded. iOS and Android both suppress Voice Wake in the background.
- **Wake word is ignored:** say the trigger, pause, then the command. A trigger with no command is ignored. Android also ignores extra words before the trigger (`tell openclaw ...`).
- **Triggered, but nothing reaches the agent:** reconnect the Gateway. Android reports **Gateway unavailable** when the `voice.transcript` event cannot be sent.
- **Wake-word edits do not stick on Android:** connect first, then tap **Save wake words**.
- **False triggers:** shorten the list. Keep phrases unique. The Gateway list is shared with macOS and every other node.

## Related

- [Voice wake](/nodes/voicewake)
- [Voice wake (macOS)](/platforms/mac/voicewake)
- [Talk mode](/nodes/talk)
- [iOS app](/platforms/ios)
- [Android app](/platforms/android)
