---
name: create-widget
description: Scaffold a new Aperture Live widget page from widget-starter.html, wire up the commands and events the developer asks for, and produce the matching metadata.widgets entry for the session configuration. Use when someone wants to build, start, or generate an Aperture Live widget.
---

# Create an Aperture Live widget

## 1. Load the contract

Load the `widget-bridge-reference` skill first. Use only the commands and events it lists. If the developer asks for something the bridge can't do, such as reading the mute state, picking a specific camera, or a new event, say so and offer the closest supported option.

## 2. Ask what's missing

Get these from the developer. Ask only for what you can't infer, and fill the rest with the defaults.

| Input | Default |
|---|---|
| File name and location | `widgets/<slug>.html` |
| Controls and features (see the bridge reference) | Toggle mic and flip camera |
| Whether it supplies pending values, and which headers | No |
| The session's transport: WHIP, RTMP or SRT (ask only if it supplies pending values) | WHIP |
| The HTTPS URL where the page will be hosted | `https://widgets.example.com/<slug>.html`, marked as a placeholder |
| Whether it must work on Android | No |

## 3. Write the page

1. Copy `widget-starter.html` to the target path.
2. Keep the `AppBridge` block as it is.
3. Replace the UI section and the widget logic:
   - Use one button for each command, with at least 44 × 44 pt touch targets, and readable text over a translucent dark background.
   - Offer camera switching as a single "Flip camera" button (`flipCamera`). `userCamera` and `worldCamera` currently toggle, so a "Front"/"Back" pair would be misleading. Add that pair only if the developer insists, and leave a comment that explains the toggle.
   - Show the stream state from `callStateChanged`. Start the display at "unknown", and don't run logic that assumes a state until the first event arrives. The app sends no state on load, and the widget can load before the broadcast starts (pending values, test widget).
   - For snapshots, send `requestSnapshotImage`, show the `snapshotImage` data URL in an `<img>`, and add a timeout message, since the app may not reply.
   - Pending values only work for WHIP sessions; for RTMP or SRT the app goes live without them. If the session uses RTMP or SRT, don't build a pending-values flow. Tell the developer to put the credentials in the configuration (`rtmpStreamKey`, `srtStreamKey`, `srtPassphrase`) or to fetch a fresh configuration with `setSessionConfigurationLink`.
   - For pending values, send `setPendingValues` with `{ headers: { … } }` once the page has the values. Make every value a string, or every value an array of strings; never mix them. Never use the headers the app drops (see the reference). Validate the payload before sending: the app goes live after the first `setPendingValues`, even if the headers were unusable.
   - For `updateSessionConfiguration`, send the complete configuration, not just the changed fields; settings left out can be reset.
   - The starter already sends `setWebviewOrigin` with `location.origin` on load; keep it. Remove it only if the widget navigates to a page on another origin, such as a sign-in redirect: the app keeps the first origin, so events would stop reaching the new page.
   - For `sessionConfigurationUpdated`, call `JSON.parse(data)` and show `success` or `error`.
   - If the page calls an API on another origin, such as a token endpoint, remind the developer that the API must send CORS headers for the widget's origin. Widget `headers` only go on the first page request, so explain how the page identifies itself: a query string on the widget URL, or a cookie whose `domain` is the API's host. See "Calling your own API" in `docs/widgets.md`.
4. Keep it to one self-contained file with no build step and no external scripts, unless the developer asks for them.

## 4. Produce the session configuration entry

Print one snippet, rooted at `metadata`, that the developer merges into their configuration's existing `metadata` object. Include `options.pendingValues` in the same snippet when the widget supplies pending values:

```json
{
  "metadata": {
    "widgets": [
      { "key": "appWidget", "url": "https://widgets.example.com/<slug>.html" }
    ],
    "options": {
      "pendingValues": { "headers": ["X-Stream-Token"] }
    }
  }
}
```

- Always use `key: "appWidget"` so the entry also works on Android.
- Add `headers` or `cookies` only if the page needs them. Give each cookie a `domain`.
- Leave out `options` when the widget doesn't supply pending values. If the developer's configuration already has `metadata.options`, tell them to merge `pendingValues` into it rather than replace it.
- If the developer gave you a whole configuration, merge the snippet into it and validate the result with the `validate-session-config` skill.
- If you produced only the snippet, still validate it with `validate-session-config`, but run only the checks that apply to a fragment: widgets, cookies, `pendingValues` and retired keys. Tell the developer that the endpoint and header checks were skipped because the snippet has no ingest settings.

## 5. Check before you finish

- Every `send(...)` type and every `on(...)` name appears in the bridge reference.
- Every message goes through `window.webkit.messageHandlers.nativeApp`; there's no `http://` widget URL, and no real secrets in the page or the configuration.
- Open the page in a desktop browser if you can: it must load without errors and log `[AppBridge] no bridge; would send …` when you click a button.

Then tell the developer how to test it on a device. Point them to "Testing your widget" in `docs/widgets.md`: host over HTTPS (a tunnel works), put the configuration URL in a QR code, scan it, and go live.
