---
name: validate-session-config
description: Check an Aperture Live session configuration JSON document against session-configuration-schema.yaml and the app's runtime rules, and report each problem with its JSON path. Use when someone shares a session configuration or QR payload, or asks whether one is valid.
---

# Validate a session configuration

`session-configuration-schema.yaml` is the source of truth for field names, types, and enums. Read it before you check anything, and don't rely on memory.

## How the app reads a configuration

- Key names are matched exactly as the schema spells them. `rtmpURL` isn't `rtmpUrl`, and `ice_servers` isn't `iceServers`. A misspelled key isn't an error; the app ignores it without a warning. Report keys that are close to a real key as likely typos.
- Integer options of the wrong type, such as `"41"` instead of `41`, are dropped one at a time; the rest of the configuration still loads.
- String enums are matched case-insensitively. A value that isn't in the enum falls back to the default.

## Checks

Report each finding as **error** (the configuration is rejected, or the feature won't work), **warning** (likely a mistake), or **note**, together with its JSON path:

1. The document parses as JSON, and the top level is an object.
2. At least one ingest endpoint is present: `whip`, `metadata.rtmpURL` or `metadata.srtURL` (error if none). An empty or whitespace-only `rtmpURL` / `srtURL` doesn't count. `whip`, when present, uses `https://` (error otherwise).
   - When more than one is set, note which one the app uses: RTMP first, then SRT, then WHIP. The others are ignored.
   - If there's no `whip`, or no `ice_servers` / `stun_urls` entry, warn that the Android app rejects the configuration; Android accepts only WHIP payloads with ICE servers.
3. For RTMP (`metadata.rtmpURL` present and non-empty), the URL is `rtmp://` or `rtmps://`. `rtmpStreamKey` is used only together with it.
4. For SRT (`metadata.srtURL` present and non-empty), each of these is an error, because the stream fails to connect:
   - `srtURL` doesn't use `srt://`, has no host, or contains more than one `?`. A second `?` usually means the stream ID was pasted into the URL; it belongs in `srtStreamKey`.
   - `srtStreamKey` is longer than 512 bytes.
   - `srtPassphrase` is set but isn't 10–79 characters.
   - `srtLatencyMs` is 0 or negative.

   Note any other `srt*` field set without `srtURL`; it has no effect.
5. `metadata` fields have the right JSON type. A wrong type there, such as `"srtLatencyMs": "120"` (a string), fails the whole payload, unlike integer fields in `metadata.options`, which are dropped one at a time.
6. `headers.Authorization`, if present: the app removes every case-sensitive `"Bearer "` and uses the rest as the token, so `Bearer <token>` and a bare `<token>` both work. Report an error for any other casing such as `bearer <token>` or `BEARER <token>`; it's sent as part of the token, and authentication fails. The app uses no other top-level header.
7. Every field's type and enum value matches the schema.
8. `ice_servers[*].urls` is an array, not a single string.
9. Widgets, at `metadata.widgets`, not at the top level:
   - Each `url` uses `https://` (error).
   - Only the first entry loads; note any others.
   - `key` should be `"appWidget"` for Android support (warning).
   - Each cookie has `name` and `value` (error) and should have `domain` (warning; Android skips cookies without one).
10. `metadata.options.pendingValues`, if present, is a non-empty object whose values are arrays of strings, and there is a widget to supply the values (error if there's no widget). If the session uses RTMP or SRT (`metadata.rtmpURL` or `metadata.srtURL` is set), report an error: pending values are WHIP-only, so the app waits for the widget and then goes live without the values.
11. Deprecated field: `stun_urls` (use `ice_servers`).
12. Retired keys that iOS still decodes: `metadata.privacy`, `metadata.location`, `metadata.ptt`, `metadata.locationHref`, `metadata.options.privacy`, `metadata.options.ui.hideBar` and `metadata.options.ui.contentMargin`. They're not in the schema and have no effect, but iOS still type-checks them. Report a warning to remove any that are present. Report an error if one has the wrong type, because iOS rejects the whole payload: the `privacy`, `location` and `ptt` keys and `hideBar` must be booleans, `locationHref` a string, and `contentMargin` an integer.

## Output

Start with one line saying whether the app will accept the configuration, then list the findings, errors first. If there are fixes, offer a corrected copy of the JSON. Never print token or credential values back in full; shorten them, for example `Bearer eyJ…`.
