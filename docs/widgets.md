# Building an Aperture Live widget

A widget is a web page that Aperture Live shows on top of the camera view during a broadcast. Your page runs in the app's web view and talks to the app over a small JavaScript bridge. With it, the page can mute the microphone, flip the camera, take a snapshot of the camera feed, and hand the app values it needs before it goes live.

This guide covers everything you need to build one:

1. [How a widget gets into the app](#how-a-widget-gets-into-the-app)
2. [Adding a widget to a session configuration](#adding-a-widget-to-a-session-configuration)
3. [The bridge](#the-bridge)
4. [Command reference (widget → app)](#command-reference-widget--app)
5. [Event reference (app → widget)](#event-reference-app--widget)
6. [Flows](#flows): snapshot, pending values, session configuration updates
7. [Layout and styling](#layout-and-styling)
8. [Testing your widget](#testing-your-widget)
9. [Platform differences](#platform-differences)
10. [Troubleshooting](#troubleshooting)

To start from working code, copy [`widget-starter.html`](../widget-starter.html). For a page that exercises every command, see [`sample_widget_controls.html`](../sample_widget_controls.html).

## How a widget gets into the app

1. The broadcaster scans a QR code, or opens a deep link, that holds an **HTTPS URL**.
2. The app makes a `GET` request to that URL. The response is a session configuration JSON document. Its format is defined in [`session-configuration-schema.yaml`](../session-configuration-schema.yaml).
3. If the configuration has a `metadata.widgets` array, the app loads the **first** widget in it. Any other entries are ignored.
4. On iOS the widget panel opens when the stream goes live. If the configuration has `metadata.options.pendingValues`, the panel opens earlier, before the broadcast starts (see [Pending values](#pending-values)).
5. The broadcaster can show and hide the panel with the widget button in the streaming controls. Hiding the panel doesn't reload your page; it keeps running in the background.
6. When the stream ends or the session is reset, the app unloads the widget.

## Adding a widget to a session configuration

Widgets live in `metadata.widgets`. Here is a minimal session configuration that streams over WHIP and shows one widget:

```json
{
  "whip": "https://ingest.example.com/whip/my-stream",
  "headers": { "Authorization": "Bearer <session-token>" },
  "metadata": {
    "name": "Studio A",
    "widgets": [
      {
        "key": "appWidget",
        "url": "https://widgets.example.com/controls.html",
        "headers": { "X-Widget-Session": "abc123" },
        "cookies": [
          { "name": "session", "value": "abc123", "domain": "widgets.example.com", "path": "/" }
        ]
      }
    ]
  }
}
```

| Field | Required | What it does |
|---|---|---|
| `url` | yes | The page to load. It must use `https://`. |
| `key` | no | An identifier for the widget. Set it to `"appWidget"`, which the Android app requires (see [Platform differences](#platform-differences)). |
| `headers` | no | HTTP headers sent with the **first** page request only. Requests your page makes later don't carry them, so use cookies, or have your page store what it needs. |
| `cookies` | no | Cookies the app sets in the web view before the page loads. The app uses `name`, `value`, `domain`, and `path`. Android also applies `sameSite` and `secure`; iOS ignores them. `path` defaults to `/`. Always set `domain`: iOS falls back to the widget URL's host when it's missing, but Android skips the cookie. |

### HTTPS only

App Store and TestFlight builds refuse to load a widget URL that doesn't use `https://`. The panel stays empty and no error appears on screen. Serve your page over HTTPS from the start, even while you develop it (see [Testing your widget](#testing-your-widget)).

## The bridge

Messages go both ways:

| Direction | How |
|---|---|
| Widget → app | `window.webkit.messageHandlers.nativeApp.postMessage({ type, data })` |
| App → widget | The app calls `window.postMessage({ name, data }, targetOrigin)` inside your page. Listen with `window.addEventListener('message', …)`. |

### Sending a command

```js
function send(type, data) {
  const handler = window.webkit?.messageHandlers?.nativeApp;
  if (!handler) {
    console.warn('Not running inside Aperture Live; dropped', type);
    return false;
  }
  handler.postMessage({ type, data: data ?? null });
  return true;
}

send('toggleMic');
```

- `type` is required. A message without a string `type` is ignored.
- `data` depends on the command. Send `null` for commands that take no payload.
- Look up the handler each time you send, as above, rather than once at page load. On Android the bridge is installed as the page starts loading, so a check made very early can miss it.
- Most commands are fire-and-forget. Only three send a reply: `requestSnapshotImage` replies with `snapshotImage`, and the two session configuration commands reply with `sessionConfigurationUpdated`.
- The app ignores unknown `type` values without an error, so a typo fails silently. Copy command names from the reference below.

### Receiving events

```js
window.addEventListener('message', (event) => {
  const msg = event.data;
  if (!msg || typeof msg.name !== 'string') return; // not from the app

  switch (msg.name) {
    case 'callStateChanged':
      console.log('Stream state:', msg.data); // e.g. "live"
      break;
    case 'snapshotImage':
      document.querySelector('#snapshot').src = msg.data;
      break;
    case 'sessionConfigurationUpdated': {
      const result = JSON.parse(msg.data); // { success, error? }
      console.log(result);
      break;
    }
  }
});
```

- `event.data.data` is **always a string**. Events that carry structured data, such as `sessionConfigurationUpdated`, send it as a JSON string, so call `JSON.parse` on it.
- Your page also receives any `message` events that it or its iframes post. Check `name` before you act on a message.
- By default the app posts with target origin `'*'`. To narrow it, send [`setWebviewOrigin`](#setwebvieworigin).

## Command reference (widget → app)

Every command is a `{ type, data }` object sent through `nativeApp.postMessage`.

| `type` | `data` | What the app does |
|---|---|---|
| `toggleMic` | `null` | Toggles the microphone mute. |
| `mute` | `null` | Mutes the microphone. Does nothing if it's already muted. |
| `unmute` | `null` | Unmutes the microphone. Does nothing if it's already unmuted. |
| `flipCamera` | `null` | Switches between the front and back cameras. |
| `userCamera` | `null` | Meant to select the front (selfie) camera. **Currently toggles the camera, like `flipCamera`.** |
| `worldCamera` | `null` | Meant to select the back (world-facing) camera. **Currently toggles the camera, like `flipCamera`.** |
| `startBroadcasting` | `null` | Starts the stream, like tapping Go Live. If the profile has no ingest URL, the app opens the QR scanner instead. |
| `stopBroadcasting` | `null` | Shows the app's stop confirmation. The stream keeps going until the broadcaster confirms. |
| `requestSnapshotImage` | `null` | Asks for a JPEG of the current camera frame. The reply is a [`snapshotImage`](#snapshotimage) event. |
| `setPendingValues` | `{ headers: { … } }` | Supplies the values the app is waiting for before it goes live. See [Pending values](#pending-values). |
| `setWebviewOrigin` | origin string, such as `"https://widgets.example.com"` | Sets the target origin the app uses when it posts events to your page. |
| `setSessionConfigurationLink` | `{ url, otp? }` | Fetches a new session configuration from `url` and applies it. The reply is a [`sessionConfigurationUpdated`](#sessionconfigurationupdated) event. |
| `updateSessionConfiguration` | a complete session configuration object | Saves the object to the active profile. Settings you leave out may be reset, so send the whole configuration (see [Session configuration updates](#session-configuration-updates)). The reply is a [`sessionConfigurationUpdated`](#sessionconfigurationupdated) event. |

The app doesn't report the mute state or which camera is active. If your UI shows them, track them yourself, and keep in mind the broadcaster can also change them with the app's own controls. Because `userCamera` and `worldCamera` toggle for now, a button labeled "Front camera" can select the back camera. Until they select a specific camera, offer a single "Flip camera" button.

### `setWebviewOrigin`

```js
send('setWebviewOrigin', window.location.origin);
```

After this, the app delivers events only while your page's origin matches this value. If you set a different origin, events stop arriving without an error. Send it once, after the page loads. The app keeps the value until the widget is unloaded, so don't send it from a widget that navigates to a page on another origin, such as a sign-in redirect; events would stop reaching that page.

## Event reference (app → widget)

Each event arrives as a `message` event whose `event.data` is `{ name, data }`, with `data` a string.

### `callStateChanged`

Sent each time the stream state changes. `data` is one of these strings:

| Value | Meaning |
|---|---|
| `idle` | Not streaming. |
| `creating` | The app is setting up the stream. |
| `connecting` | The app is connecting to the ingest server. |
| `live` | On air. |
| `reconnecting` | The connection dropped and the app is retrying. |
| `failed` | The stream failed. |
| `stopped` | The stream ended. |

The app doesn't send the current state when your page loads; you receive the next change. Treat the state as **unknown** until the first event arrives. The widget usually loads just after the stream goes live, but it also loads before the broadcast starts while the app waits for [pending values](#pending-values), and when the built-in test widget is opened. Don't assume `live` on load. Write the code so it also accepts states it hasn't seen before.

### `snapshotImage`

The reply to `requestSnapshotImage`. `data` is a JPEG data URL (`data:image/jpeg;base64,…`) that you can set straight into an `<img src>`.

### `sessionConfigurationUpdated`

The reply to `setSessionConfigurationLink` and `updateSessionConfiguration`. `data` is a JSON string:

On success:

```json
{ "success": true }
```

On failure:

```json
{ "error": "URL host not allowed", "success": false }
```

`error` is a human-readable message for logs. Don't compare it to exact strings in your code, because the wording can change and differs between platforms.

## Flows

### Snapshot

```js
send('requestSnapshotImage');
// later, in your message listener:
// msg.name === 'snapshotImage' → img.src = msg.data
```

The app compresses the JPEG so it fits through the bridge. If no camera frame is available yet, the app drops the request and sends nothing, so don't wait for the reply forever. Use a timeout.

### Pending values

Use pending values when the stream can't start until the widget supplies something, such as a token the page obtains after the broadcaster signs in.

1. Put a non-empty `metadata.options.pendingValues` object in the session configuration. The app checks only that it isn't empty; the keys list what your widget will supply.

   ```json
   "options": {
     "pendingValues": { "headers": ["X-Stream-Token"] }
   }
   ```

2. When the broadcaster accepts the session, the app opens your widget **before** going live and waits.
3. Your widget sends `setPendingValues`:

   ```js
   send('setPendingValues', {
     headers: { 'X-Stream-Token': token }
   });
   ```

   Header values are either **all strings** or **all arrays of strings**; the app joins array values with commas. Don't mix the two in one message: iOS then discards every header (see the rules below).
4. The app adds the headers to the request it sends to the ingest server, then starts the broadcast right away.

Rules:

- The app accepts `setPendingValues` only while it's waiting. At any other time it ignores the command.
- The app drops the headers `Authorization`, `Content-Type`, `Content-Length`, `Host`, `Cookie`, and `Connection`, in any letter case. Put the session token in the configuration's top-level `headers` instead.
- If `data` isn't an object, the app ignores the command and keeps waiting. The broadcaster can cancel or scan again.
- **You get one try.** If `data` is an object but `headers` is missing, isn't an object, mixes strings and arrays, or contains only dropped headers, the app still stops waiting and starts the broadcast, without your values. The app doesn't tell the widget, and it won't accept another `setPendingValues`. Check your payload before you send it: `{ headers: { … } }`, with every value a string, or every value an array of strings.

### Session configuration updates

Two commands change the session configuration from inside the widget. Both reply with `sessionConfigurationUpdated`.

**`setSessionConfigurationLink`** fetches a new configuration, for example to renew credentials that are about to expire:

```js
send('setSessionConfigurationLink', {
  url: 'https://config.example.com/session/abc123',
  otp: '123456' // optional; when present, the app submits it to `url` instead of a plain GET
});
```

- `url` must use `https://`, and its host must be the same as the host of the QR code or link that set up the session. The app rejects any other host.
- The app applies the new configuration to the current session. If the stream is `live`, `connecting` or `creating`, the app stops it and starts it again with the new configuration.
- While the stream is `reconnecting`, the app saves and applies the new configuration but doesn't restart, so the reconnect attempt that's already running keeps the old credentials. You still get `success: true`. Avoid renewing credentials during `reconnecting`; wait for `live`, or for `failed` and then a new start.

**`updateSessionConfiguration`** sends the configuration itself. **Send a complete configuration**, the same document your QR code URL returns, with your changes applied:

```js
send('updateSessionConfiguration', {
  whip: 'https://ingest.example.com/whip/my-stream',
  headers: { Authorization: 'Bearer <session-token>' },
  ice_servers: [{ urls: ['stun:stun.example.com:3478'] }],
  metadata: {
    name: 'Studio A',
    widgets: [{ key: 'appWidget', url: 'https://widgets.example.com/controls.html' }],
    options: { resolution: '1080p', videoBitrate: 4000 }
  }
});
```

- The object has the same shape as a configuration fetched from a QR code, and the app reads it with the same rules (see [`session-configuration-schema.yaml`](../session-configuration-schema.yaml)). It must include an ingest endpoint: `whip`, `metadata.rtmpURL` or `metadata.srtURL`.
- **Don't send a partial update.** For settings you leave out, the app keeps only `whip`, the `Authorization` token, `widgets`, `ui`, `pendingValues`, `logUrl`, `lockOrientation`, `lockCameraMode`, `startBitrate`, and the H.264, `useCBR`, `fixedEncodingMode` and `audioPriorityMode` options. Everything else resets to its default or disappears: the ICE servers, the display name (`metadata.name`), `resolution`, `videoBitrate`, `audioBitrate`, `videoCodec`, `framerate`, `defaultCameraFacing`, `streamName`, `platformName`, and the RTMP and SRT fields. For example, an update that sends only `{ whip, metadata: { options: { resolution: '1080p' } } }` wipes the ICE servers and display name, and switches an RTMP or SRT profile to WHIP.
- A setting that you include replaces the stored one as a whole; for example, sending `widgets` replaces the whole widget list.
- The app saves the change to the profile only. It doesn't restart a running stream, and it doesn't apply the change to the current session either, so a `startBroadcasting` sent straight afterwards may still use the old settings. To change settings for the current session, use `setSessionConfigurationLink`, which applies them right away.
- iOS supports this command. Android doesn't yet (see [Platform differences](#platform-differences)).

## Layout and styling

- The widget panel covers the camera view, inset from the screen edges by a fixed amount. The app sets its size, which changes between iPhone, iPad, and orientations, so build a responsive layout.
- The web view is transparent, so the camera shows through any part of your page without a background. A translucent dark background, such as `rgba(0, 0, 0, 0.6)`, keeps text readable over video.
- Add `<meta name="viewport" content="width=device-width, initial-scale=1">`. The page can scroll vertically, but scroll bars and bounce are turned off, so a layout that fits without scrolling works best.
- Make touch targets at least 44 × 44 points. Broadcasters often tap while they hold the device up.
- Session-wide layout options live in `metadata.options.ui`. Widgets have no layout keys of their own.

  | Option | Effect |
  |---|---|
  | `compact` | `true` uses the app's compact control layout, which leaves more room for the widget. It overrides the broadcaster's own compact-mode setting. |

## Testing your widget

### In a desktop browser

Open your page in a browser. The bridge isn't there, so `send()` returns `false`. You can still build the layout, and you can fake app events in the console:

```js
window.postMessage({ name: 'callStateChanged', data: 'live' }, '*');
```

The starter template logs each command it would have sent, so you can check them in the console.

### In the app

1. Host your page at an `https://` URL that the device can reach. For local development, an HTTPS tunnel to your machine works, such as `ngrok http 8080` or `cloudflared tunnel --url http://localhost:8080`.
2. Host a session configuration JSON document at another `https://` URL and point its `metadata.widgets[0].url` at your page. The configuration also needs a working ingest destination: a WHIP endpoint (`whip`), an RTMP ingest (`metadata.rtmpURL`, plus `metadata.rtmpStreamKey` when needed), or an SRT ingest (`metadata.srtURL`, plus `metadata.srtStreamKey` when needed). An RTMP ingest from a streaming platform works well for testing. RTMP and SRT configurations work only on iOS for now; Android needs a WHIP configuration.
3. Make a QR code that holds the configuration URL, and scan it in Aperture Live. Accept the session and go live. The widget opens when the stream is live.
4. Tap your buttons. A widget that mutes the mic and flips the camera is enough to confirm the bridge works.

### Inspecting the page on iOS

Aperture Live allows Web Inspector on its widget web view. Connect the device to a Mac, turn on **Settings → Apps → Safari → Advanced → Web Inspector** on the device, then open Safari on the Mac and pick the device and your page from the **Develop** menu. Use it to read `console` output and send test commands from the console.

### The built-in test widget (iOS)

Aperture Live ships with a test widget that shows the stream state, the last event received, and a button for each control command. Turn on **Debug Tools** in the app's preferences, then use **Load Test Widget** in Profile Settings. It works without scanning a QR code, which makes it a quick way to tell whether a problem is in your page or in the app.

## Platform differences

The bridge described above is the iOS bridge. The Android app accepts the same `webkit.messageHandlers.nativeApp.postMessage(...)` call and sends the same events, so you can use one page for both. The Android app's widget support is more limited:

| Behavior | iOS | Android |
|---|---|---|
| Which widget loads | The first entry in `metadata.widgets` | The entry whose `key` is `"appWidget"` |
| When the widget can be used | While live, and while waiting for pending values | Only while waiting for pending values |
| `setPendingValues`, `setWebviewOrigin`, `setSessionConfigurationLink` | Supported | Supported |
| Control commands (`toggleMic`, `mute`, `unmute`, `flipCamera`, `userCamera`, `worldCamera`, `startBroadcasting`, `stopBroadcasting`) | Supported | Ignored |
| `requestSnapshotImage`, `updateSessionConfiguration` | Supported | Ignored |
| `callStateChanged` | Sent | Sent, with the same values |
| `sessionConfigurationUpdated` errors | A message for each failure | `"refresh_failed"` |
| Cookie without `domain` | Uses the widget URL's host | Skipped |

To work on both platforms, set `key: "appWidget"`, give every cookie a `domain`, and assume commands other than those listed as supported on Android may do nothing there.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The panel is empty and the page never loads | The URL isn't `https://`, the server returned an error, or the device can't reach the host. |
| Buttons do nothing | The page posts through something other than `window.webkit.messageHandlers.nativeApp`, or a `type` is misspelled. |
| No events arrive | `setWebviewOrigin` was set to an origin that doesn't match the page, or the listener ignores the message shape `{ name, data }`. |
| `JSON.parse` fails on `callStateChanged` | Its `data` is a plain string such as `live`, not JSON. Only `sessionConfigurationUpdated` sends JSON. |
| No `snapshotImage` reply | No camera frame was available yet. Retry after the preview is running. |
| "Front camera" switches to the back camera | `userCamera` and `worldCamera` currently toggle. See the [command reference](#command-reference-widget--app). |
| `setPendingValues` is ignored | The app isn't waiting for pending values: the configuration has no `metadata.options.pendingValues`, or the app already accepted an earlier `setPendingValues`. |
| The stream starts, but without your pending values | `headers` was missing, wasn't an object, mixed strings and arrays, or held only dropped headers. The app went live without them. See the [pending values rules](#pending-values). |
| `setSessionConfigurationLink` replies `URL host not allowed` | The host of `url` isn't the host of the QR code or link that set up the session. |
| Cookies are missing on Android | The cookie has no `domain`. |
