---
name: widget-bridge-reference
description: The exact Aperture Live widget JavaScript bridge contract (command types, payloads, events, platform limits). Load before writing, editing, reviewing, or explaining any widget code that talks to the Aperture Live app, so no message type or payload is invented.
---

# Aperture Live widget bridge contract

This is the complete contract. If a widget needs something that isn't listed here, the app can't do it. Say so instead of making up a message type, field, or event.

The full guide, with examples and the reasons behind each rule, is `docs/widgets.md`. Read it when you need more than this summary.

## Transport

- **Widget → app:** `window.webkit.messageHandlers.nativeApp.postMessage({ type: string, data: any })`. Look up the handler on each send; don't cache it at load.
- **App → widget:** the app calls `window.postMessage({ name: string, data: string }, targetOrigin)`. Listen with `window.addEventListener('message', e => …)` and read `e.data.name` and `e.data.data`.
- Event `data` is **always a string**. Only `sessionConfigurationUpdated` carries JSON that needs `JSON.parse`.
- The app ignores unknown command types without an error.
- `window.webkit.messageHandlers.nativeApp` is the only bridge, on iOS and Android. Don't post through any other object.

## Commands (widget → app)

| `type` | `data` | Effect |
|---|---|---|
| `toggleMic` | `null` | Toggles the mic mute |
| `mute` | `null` | Mutes the mic (does nothing if already muted) |
| `unmute` | `null` | Unmutes the mic (does nothing if already unmuted) |
| `flipCamera` | `null` | Switches between the front and back cameras |
| `userCamera` | `null` | Currently toggles, like `flipCamera` |
| `worldCamera` | `null` | Currently toggles, like `flipCamera` |
| `startBroadcasting` | `null` | Goes live (opens the QR scanner if there is no ingest URL) |
| `stopBroadcasting` | `null` | Shows the native stop confirmation |
| `requestSnapshotImage` | `null` | The reply is `snapshotImage`, or nothing if no frame is available |
| `setPendingValues` | `{ headers: { [name]: string } }` or `{ headers: { [name]: string[] } }`, never mixed | Accepted only while the app waits for pending values. A `data` value that isn't an object is ignored, and the app keeps waiting. Any object, including one whose `headers` is missing, mixed or all dropped, makes the app go live, and no second try is accepted |
| `setWebviewOrigin` | origin string | Sets the target origin for events; use `location.origin` |
| `setSessionConfigurationLink` | `{ url: string, otp?: string }` | `url` must be HTTPS on the host of the session's QR code. Applied to the current session, and restarts the stream if it's `live`, `connecting` or `creating`, but not while `reconnecting`. The reply is `sessionConfigurationUpdated` |
| `updateSessionConfiguration` | a **complete** session configuration object (must include `whip`, `metadata.rtmpURL` or `metadata.srtURL`) | Saved to the profile only: not applied to the current session, and no restart. Left-out settings such as ICE servers, display name, video settings and RTMP/SRT fields are reset, so never send a partial update. The reply is `sessionConfigurationUpdated` (iOS only) |

`setPendingValues` drops these headers, matched in any letter case: `Authorization`, `Content-Type`, `Content-Length`, `Host`, `Cookie`, `Connection`.

Only three commands reply: `requestSnapshotImage` (`snapshotImage`), `setSessionConfigurationLink` and `updateSessionConfiguration` (both `sessionConfigurationUpdated`).

## Events (app → widget)

| `name` | `data` |
|---|---|
| `callStateChanged` | `idle` \| `creating` \| `connecting` \| `live` \| `reconnecting` \| `failed` \| `stopped` |
| `snapshotImage` | `data:image/jpeg;base64,…` |
| `sessionConfigurationUpdated` | JSON string `{"success":bool,"error"?:string}` |

No other events exist. The app reports neither the mute state nor the active camera, and it doesn't send the current state when the page loads. Treat the state as unknown until the first `callStateChanged`. The widget usually loads just after going live, but it also loads before the broadcast while waiting for pending values.

## Session configuration entry

```json
"metadata": {
  "widgets": [
    { "key": "appWidget", "url": "https://…", "headers": { }, "cookies": [ { "name": "", "value": "", "domain": "", "path": "/" } ] }
  ]
}
```

- `url` must be `https://`.
- Only the first widget loads. Android requires `key: "appWidget"`.
- `headers` go on the first page request only.
- Cookies use `name`, `value`, `domain`, and `path`. Always set `domain`.
- Layout is session-wide in `metadata.options.ui`, whose only option is `compact`.

## Android

Android installs the same `webkit.messageHandlers.nativeApp` shape. It acts only on `setPendingValues`, `setWebviewOrigin`, and `setSessionConfigurationLink`, and the widget is usable only while the app waits for pending values. It still sends `callStateChanged`. When a widget needs controls that work on Android, say that the controls are iOS-only today.
