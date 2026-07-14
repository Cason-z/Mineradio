# LX Custom Source and VIP Playback Design

Date: 2026-07-14
Status: Approved for implementation planning

## Goal

Add LX Music desktop custom-source script support to Mineradio without replacing Mineradio's existing NetEase and QQ search, account, lyric, playlist, or playback fallback systems.

The feature resolves playable audio URLs for songs already found by Mineradio. It must improve the handling of VIP, trial-only, and unavailable songs while preserving reliable built-in playback for ordinary songs.

## Confirmed Product Decisions

- Run LX scripts in a dedicated hidden Electron `BrowserWindow`.
- Import scripts only from local `.js` files.
- Keep at most 20 imported scripts.
- Allow only one active LX source at a time.
- Put source selection and source management inside the existing quality popover.
- Keep the quality control visible in both simple and DIY modes so the source entry is always reachable.
- Use this playback order: built-in full URL, active LX source, built-in trial URL, cross-platform fallback.
- When LX resolves a song, show one short notice and mark the current quality label with `LX`.
- Allow LX network requests to public, localhost, and LAN HTTP/HTTPS endpoints for upstream compatibility.
- Do not add audio output device selection.

## Scope

### Included

- LX custom-source API version `2.0.0` compatibility needed by desktop scripts.
- Local script import, validation, listing, selection, deletion, and persistence.
- NetEase (`wy`) and QQ (`tx`) `musicUrl` actions.
- Advertised LX qualities: `128k`, `320k`, `flac`, and `flac24bit`.
- VIP/trial interception and deterministic fallback behavior.
- Playback-source feedback in the existing player controls.
- Runtime isolation, timeout handling, cancellation, crash recovery, and failure suppression.

### Not Included

- Online URL import.
- Automatic script updates.
- Audio output device selection.
- LX search support.
- New search providers for Kuwo, Kugou, or Migu.
- LX `local` source handling for local music, lyric, or picture actions.
- Replacing Mineradio's built-in lyric, cover, playlist, login, or account logic.

## Existing System Constraints

Mineradio currently treats NetEase and QQ as the only online providers. Search results are mapped in `server.js`, while `public/index.html` selects the provider-specific URL and lyric endpoints. The local HTTP server is required from Electron's main process, so the server and the source runtime can share one source-manager module without a second process.

The current NetEase URL resolver may return a trial URL. The frontend treats any URL as success, which prevents source fallback. The new resolver must retain a trial response as a fallback candidate instead of playing it immediately.

The audio proxy currently applies a NetEase referer to unknown hosts. LX-resolved URLs must be proxied without forcing a NetEase referer unless the URL host is known to require the existing NetEase or QQ behavior.

## Architecture

### SourceManager

Electron main-process module responsible for:

- importing UTF-8 `.js` files through a native file picker;
- parsing `@name`, `@description`, `@version`, `@author`, and `@homepage` metadata;
- rejecting duplicate, malformed, or scripts larger than 2 MiB;
- persisting script metadata, raw script content, and the active source ID;
- enforcing the 20-script limit;
- starting, switching, and stopping the active runtime;
- exposing source list and source status through narrow IPC handlers.

Source files live under:

```text
app.getPath('userData')/lx-sources/
  index.json
  scripts/<source-id>.js
```

Metadata writes use a temporary file plus rename. A failed import or failed initialization must not replace the current active source.

### SourceRuntime

One hidden `BrowserWindow` runs the active script. Switching sources destroys the previous runtime before creating the next one.

Runtime restrictions:

- `contextIsolation: true`
- `nodeIntegration: false`
- `nodeIntegrationInWorker: false`
- restrictive CSP
- deny navigation, redirects, WebViews, new windows, dialogs, and permissions
- no direct filesystem or Node access from the script world
- ignore `openDevTools` in packaged builds

The preload exposes the compatible subset of `globalThis.lx`:

- `version: '2.0.0'`
- `env: 'desktop'`
- `EVENT_NAMES`
- `on`, `send`, and callback-based `request`
- `currentScriptInfo`
- buffer, crypto, and zlib helpers used by desktop source scripts

`lx.request` accepts only HTTP and HTTPS URLs. It intentionally allows localhost and LAN endpoints. The import UI must warn that an imported script can access the network and should come from a trusted source.

`updateAlert` may produce a passive update notice and open the supplied webpage after explicit user confirmation. It must never download or replace a script automatically.

### PlaybackResolver

A server-side facade accepts a Mineradio song and requested Mineradio quality. It owns provider mapping, quality mapping, the built-in/LX/trial order, result normalization, and error classification.

The frontend continues to request one song URL. It receives normalized metadata describing whether the final URL came from a built-in resolver, LX, a trial, or cross-platform fallback.

## Song Mapping

NetEase maps to LX source `wy`:

```js
{
  source: 'wy',
  songmid: song.id,
  name: song.name,
  singer: song.artist,
  albumName: song.album,
  img: song.cover,
  interval: formattedDuration,
  types,
  _types,
  typeUrl: {}
}
```

QQ maps to LX source `tx`:

```js
{
  source: 'tx',
  songmid: song.mid,
  songId: song.qqId,
  strMediaMid: song.mediaMid,
  albumMid: song.albumMid,
  name: song.name,
  singer: song.artist,
  albumName: song.album,
  img: song.cover,
  interval: formattedDuration,
  types,
  _types,
  typeUrl: {}
}
```

Search and playlist mapping must preserve the source-specific IDs and available quality metadata needed to build these objects later.

Local files and podcasts bypass the LX resolver.

## Quality Mapping

Mineradio quality preferences map to LX qualities in descending fallback order:

| Mineradio request | LX attempts |
| --- | --- |
| `jymaster` | `flac24bit`, `flac`, `320k`, `128k` |
| `hires` | `flac24bit`, `flac`, `320k`, `128k` |
| `lossless` | `flac`, `320k`, `128k` |
| `exhigh` | `320k`, `128k` |
| `standard` | `128k` |

Only qualities advertised by the active script for the requested platform may be attempted. A source that does not advertise `wy` or `tx` is skipped for that platform.

## VIP and Playback Flow

For every online track:

1. Ask the built-in provider for the requested quality.
2. If it returns a full, non-trial URL, play it immediately.
3. If it returns a trial URL, retain the trial response without starting playback.
4. If an active LX source supports the track platform, request `musicUrl` using the mapped `musicInfo` and quality sequence.
5. If LX returns a valid URL, play it and discard the retained trial candidate.
6. If LX fails and a trial candidate exists, play the trial and show the existing trial restriction UI.
7. If no trial exists, run the existing same-title/same-artist cross-platform fallback.
8. Resolve the alternate track through the same built-in/LX/trial sequence, with fallback depth limited to one platform switch.
9. If all attempts fail, preserve the existing skip or unavailable-song behavior.

Selecting `内置` in the source selector skips steps 3 through 5 involving LX and preserves built-in behavior.

Track switches invalidate outstanding LX requests. Late responses cannot change the active track.

## User Interface

The quality popover gains a source section above the quality choices:

```text
播放音源
  内置
  <imported LX source>  ✓

管理自定义源  >
----------------
超清母带
高清臻音
无损
极高
标准
```

The existing quality control remains visible in simple and DIY modes.

The manager dialog shows:

- local import action;
- name, version, and author;
- supported platforms and qualities;
- active state;
- delete action;
- the trusted-source network warning;
- initialization and runtime error state.

Only one source can be selected. Selection persists across restarts. Deleting the active source switches to `内置` before destroying the runtime.

When LX supplies the current URL:

- show one short notice: `已通过 <source name> 获取播放地址`;
- render the quality label as `LX · SQ`, `LX · HQ`, or the matching quality label;
- expose the source name and resolved quality in the control tooltip;
- clear the LX marker on the next track if that track uses a built-in URL.

## Validation and Failure Handling

Import validation requires:

- a UTF-8 JavaScript file;
- a valid leading metadata comment;
- unique script content;
- successful runtime creation;
- registration of the request handler;
- a valid `inited` event with supported source/action/quality declarations.

Runtime behavior:

- source initialization times out after 10 seconds;
- each playback request times out after 20 seconds;
- returned music URLs must be HTTP/HTTPS strings no longer than 2048 characters;
- switching tracks cancels the host request when possible and always ignores stale responses;
- a crashed runtime is restarted once;
- three consecutive resolver failures disable the source for the current application session and fall back to built-in playback; a successful resolution resets the counter;
- an identical source and error-category notification is shown at most once every five minutes.

An unhealthy source remains imported and selected in persistent settings, but the manager shows its session-disabled state. A restart allows it to initialize again.

## Security and Licensing

The manager displays a warning before import because scripts can make arbitrary public, localhost, and LAN HTTP/HTTPS requests.

The runtime does not expose shell, filesystem, process, Electron, or unrestricted Node APIs. Script-provided navigation, popups, permissions, and file URLs are blocked.

LX Music desktop is Apache-2.0 licensed and Mineradio is GPL-3.0 licensed. If implementation code is copied or adapted from LX rather than independently implementing the documented protocol, preserve the required Apache attribution and update Mineradio's notice documentation.

## Testing

### Unit Tests

- metadata parsing and clipping;
- duplicate, invalid, and over-limit imports;
- NetEase-to-`wy` mapping;
- QQ-to-`tx` mapping, including `songId`, `strMediaMid`, and `albumMid`;
- quality fallback sequences and advertised-quality filtering;
- normalized URL validation;
- built-in/LX/trial/cross-platform ordering;
- stale request cancellation and fallback-depth enforcement.

### Runtime Integration Tests

Use deterministic fixture scripts and a local HTTP fixture server to verify:

- `globalThis.lx` initialization;
- `wy` and `tx` request payloads;
- crypto, buffer, zlib, and HTTP helper behavior;
- successful URL resolution;
- initialization failure;
- request rejection and timeout;
- malformed response rejection;
- crash restart and session disable behavior.

### Electron Smoke Tests

- import, select, delete, and restart persistence;
- source access from simple and DIY modes;
- ordinary songs continue using built-in full URLs;
- VIP/trial songs attempt LX before playing a trial;
- successful LX playback shows the notice and `LX` quality marker;
- LX failure reaches trial or cross-platform fallback without hanging;
- QQ quality fallback remains functional;
- podcasts and local files bypass LX;
- navigation, popup, permission, and file-protocol attempts are blocked.

## Acceptance Criteria

- Existing ordinary playback has no behavioral regression.
- A selected LX source can resolve `wy` and `tx` songs using the desktop custom-source protocol.
- VIP and trial-only tracks automatically try LX before accepting a trial.
- Every failure path returns to existing stable playback behavior without blocking controls.
- Users can always reach source selection from the quality popover in simple and DIY modes.
- The current playback source is visible when LX is used.
- No audio output device selection is added.
