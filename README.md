# Lumina

A plugin-driven YouTube Music client with an Apple Music-inspired player. One Flutter codebase for Android, Windows, macOS and Linux.

## Status

This is a scaffold. It was written without a Flutter SDK available, so **nothing has been compiled or run yet**. Expect a handful of small fixes on the first `flutter analyze`. The YouTube Music response parsing was written from memory of the API shape and has not been tested against live responses.

## Run it

```bash
cd lumina
flutter create --platforms=android,windows,macos,linux --project-name lumina .
rm test/widget_test.dart          # generated test references a template app
flutter pub get
flutter analyze
flutter test
flutter run -d windows            # or: android, macos, linux
```

`flutter create .` keeps the existing `lib/` and `pubspec.yaml`.

### Android (background playback + Android Auto)

1. Merge `platform_snippets/android/AndroidManifest.additions.xml` into `android/app/src/main/AndroidManifest.xml`.
2. Copy `platform_snippets/android/automotive_app_desc.xml` to `android/app/src/main/res/xml/`.
3. In `MainActivity.kt`, extend `AudioServiceActivity`:

   ```kotlin
   import com.ryanheise.audioservice.AudioServiceActivity
   class MainActivity : AudioServiceActivity()
   ```

4. Android Auto only lists Play Store apps by default. For sideloaded builds, enable *Unknown sources* in Android Auto's developer settings, or test with the Desktop Head Unit.

### macOS

Add the network client entitlement to `macos/Runner/DebugProfile.entitlements` and `Release.entitlements`:

```xml
<key>com.apple.security.network.client</key><true/>
```

### Desktop extras (optional)

- Install `yt-dlp` and put it on your `PATH`. Lumina falls back to it when the Dart extractor fails.
- Windows and Linux media keys need `audio_service_win` / `audio_service_mpris` (not wired yet).

## How it fits together

```
lib/
  core/
    models/      Track, AudioStream (+ honest quality label), Lyrics (LRC parser)
    plugin/      LuminaPlugin + capability interfaces, PluginHost (isolation, timeouts)
    playback/    PlayerController (queue, resolve, skip segments)
                 LuminaAudioHandler (Android/Apple system controls, Android Auto)
  sources/ytmusic/
    innertube_client.dart   POST to music.youtube.com/youtubei/v1 (WEB_REMIX)
    ytmusic_api.dart        search + defensive JSON parsing
    stream_resolver.dart    StreamResolver, CachingResolver, FallbackResolver
    youtube_explode_resolver.dart   default resolver (Dart only)
    ytdlp_resolver.dart             desktop fallback (shells out to yt-dlp)
  plugins/       SponsorBlock, Return YouTube Dislike, LRCLIB lyrics, Sound quality
  ui/            Now Playing (phone + desktop layouts), search, mini player
```

The part of this app that breaks when YouTube changes is stream resolution, so it sits behind `StreamResolver`. Resolvers stack: `CachingResolver(FallbackResolver([YoutubeExplodeResolver(), YtDlpResolver()]))`. Add or swap resolvers in `main.dart` without touching the player.

## Writing a plugin

Extend `LuminaPlugin` and implement the capabilities you need:

| Capability | Method | Used for |
| --- | --- | --- |
| `SegmentProvider` | `segmentsFor(track)` | SponsorBlock-style skipping |
| `LyricsProvider` | `lyricsFor(track)` | Synced or plain lyrics |
| `BadgeProvider` | `badgesFor(track, stream)` | Pills under the scrubber |
| `VisualsProvider` | `visualFor(track)` | Canvas-style loops (not rendered yet) |

Plugins also get `onTrackChanged` and `onPlaybackChanged`, which is all a Discord Rich Presence plugin needs. The host guards every call with a timeout and a try/catch, so a broken plugin can't stop playback.

## About the quality badge

YouTube Music serves lossy audio (Opus, AAC) up to 256 kbps. `AudioStream.qualityLabel` reports what the stream really is, and only says "Lossless" for FLAC, ALAC or PCM. That badge will light up for real once a source that serves lossless (a Subsonic/Navidrome server or a local FLAC library) is added as a resolver.

## What works on paper vs what's stubbed

Implemented: search, resolve-and-play with retry and prefetch, queue, shuffle, repeat, SponsorBlock skipping, synced lyrics, quality and dislike badges, Apple Music-style Now Playing with artwork-driven background, Android media session and Android Auto browse tree.

Not yet built:

- **Offline cache and downloads.** `PlayerController` takes an `AudioSourceFactory`; swap in `LockCachingAudioSource` plus a disk index.
- **Premium 256 kbps.** Needs a cookie-authenticated resolver. Anonymous requests top out around 160 kbps Opus.
- **Discord Rich Presence** (desktop plugin).
- **Canvas visuals.** The interface exists; no provider or rendering yet.
- **Artist notifications.** Poll followed artists with WorkManager (Android) or a timer (desktop) and diff their releases.
- **Home, library and account sign-in.**
- **Persisted settings and plugin toggles.** Storage is in-memory.
- **Word-level lyric highlighting.**

## Notes

- This goes against YouTube's terms of service. Expect GitHub or F-Droid distribution rather than Google Play, and expect periodic breakage when YouTube changes its internals.
- SponsorBlock data is CC BY-NC-SA 4.0 (attribution, non-commercial).
- Set your repo URL in the LRCLIB plugin's `User-Agent`.
