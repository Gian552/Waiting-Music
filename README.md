# WarteMusik – SCP:SL LabAPI Plugin

Plays **MP3 waiting music** in the lobby of a **SCP: Secret Laboratory** server.

The audio goes out through LabAPI's own speaker API (`SpeakerToy` /
`AudioTransmitter`), so **no fake player or bot is needed** – the plugin creates a
non-spatial speaker that everyone hears at the same volume, no matter where they
are standing.

## Features

- Plays `.mp3` files from a **music folder** while the server waits for players.
- Playlist is **shuffled** (or alphabetical) and **loops** – both configurable.
- Tracks are decoded on a **background thread** and the next one is always queued
  ahead of the running one, so there is no gap and no server stutter.
- Music stops automatically when the round starts and on round restart.
- **Every player can switch the music off for themselves** in
  `ESC → Settings → Server-specific`. The toggle is inserted at the **top** of
  that list.
- Fully configurable via `config.yml` (volume, folder, shuffle, labels …).

## Requirements

- SCP:SL dedicated server with **LabAPI 1.1** or newer.
- **NLayer.dll** – a pure managed MP3 decoder. It is pulled in from NuGet during
  the build and lands next to `WarteMusik.dll` in `bin/Release/`.

## Installation

```
…/AppData/Roaming/SCP Secret Laboratory/LabAPI/
├── plugins/<port>/WarteMusik.dll        <- the plugin
├── dependencies/<port>/NLayer.dll       <- the MP3 decoder (do not forget!)
└── configs/<port>/WarteMusik/
    ├── config.yml                       <- created on first start
    └── music/                           <- put your .mp3 files in here
```

`global` works instead of `<port>` as well if the same plugin should run on every
server on the machine.

If `NLayer.dll` is missing the plugin refuses to start and says so in the console
instead of failing silently on the first track.

## Music folder

- Location: `…/LabAPI/configs/<port>/WarteMusik/music/` (name configurable via
  `music_folder`). It is created automatically on startup.
- Only `*.mp3` directly inside the folder is picked up – no subfolders.
- The folder is **re-read at the start of every lobby**, so tracks can be added or
  removed without restarting the server.
- Any format MP3 files come in (mono/stereo, 44.1 kHz, 48 kHz, VBR …) is fine –
  the plugin downmixes to mono and resamples to the 48 kHz the game's voice
  system uses.

## Config

The two switches that matter are at the top of `config.yml`:

| Key | Default | Meaning |
| --- | --- | --- |
| `is_enabled` | `true` | Master switch. `false` = no lobby music at all. |
| `server_specific_toggle` | `true` | Adds the per-player on/off toggle to the server-specific settings menu. |
| `disabled_by_default` | `false` | `true` = players have to switch the music on themselves. |
| `debug` | `false` | Verbose console output. |
| `music_folder` | `music` | Folder name next to `config.yml`. |
| `shuffle` | `true` | Random order instead of alphabetical. |
| `loop_playlist` | `true` | Start over after the last track. |
| `volume` | `0.6` | `1.0` = unchanged. Above `1` distorts. |
| `hearing_range` | `10000` | Radius in metres the music is audible in. Must stay far larger than the map – see below. |
| `max_track_seconds` | `420` | Cuts off long tracks (memory safety net). `0` = no limit. |
| `start_delay_seconds` | `2` | Delay before the first track of a lobby. |
| `speaker_controller_id` | `231` | Audio channel. Only change on a collision with another plugin. |
| `server_specific_setting_id` | `7411` | Id of the group header; the toggle uses `7412`. |
| `settings_header` / `settings_label` / `settings_option_on` / `settings_option_off` / `settings_hint` | – | Texts in the settings menu. |

## The server-specific toggle

`ESC → Settings → Server-specific` shows:

```
Lobby
  Lobby music        [ On ] [ Off ]
```

The plugin puts its entries **in front of** the settings other plugins have
already defined, so they end up at the very top, and removes only its own entries
again when it is disabled.

Under the hood the choice is applied through `SpeakerToy.ValidPlayers` – players
who picked *Off* are simply skipped when the audio frames are sent, so they cost
no bandwidth either.

## Why `hearing_range` matters

`IsSpatial = false` only sets the AudioSource's `spatialBlend` to 0, i.e. the
music is mixed in 2D without panning or distance attenuation. It does **not**
disable culling: `AudioTransceiver.ClientReceiveMessage` drops every audio frame
whose speaker is further from the listener's camera than the speaker's
`maxDistance` – for non-spatial speakers as well.

The speaker of this plugin sits at the world origin, which is nowhere near where
players stand, so `hearing_range` has to stay large. At the prefab default the
music is silent everywhere on the map while the server log still looks perfectly
healthy. Since the mix is 2D anyway, a huge value has no audible downside.

## Memory note

Decoded audio is kept in RAM as 48 kHz mono float – roughly **11 MB per minute**
of music. At most two tracks are held at a time (the running one and the one
queued behind it), so with `max_track_seconds: 420` the worst case is about
150 MB. Lower that value if the server is tight on memory.

## Project structure

```
WarteMusik/
├── README.md
├── libs/                       # game DLLs (not committed)
└── WarteMusik/
    ├── WarteMusik.csproj
    ├── Plugin.cs               # LabAPI entry point, event wiring
    ├── Config.cs
    └── Features/
        ├── Mp3Decoder.cs       # MP3 -> mono 48 kHz float (NLayer + linear resampling)
        ├── MusicLibrary.cs     # scans the music folder, hands out the next track
        ├── LobbyMusicPlayer.cs # speaker + playlist state machine
        └── MusicPreferences.cs # server-specific toggle, per-player mute list
```

## Build

```bash
dotnet build WarteMusik/WarteMusik.csproj -c Release
```

Output lands in `WarteMusik/bin/Release/` (`WarteMusik.dll` + `NLayer.dll`).
`libs/` has to contain the managed assemblies of the **matching** server version.
