# CLAUDE.md

This is the public developer repo for Aperture Live, an iOS and Android live-streaming app. It holds what third parties need to integrate with the app: the session configuration schema, widget docs, and sample widgets. There's no app source and no build here.

## Files

| Path | What it is |
|---|---|
| `session-configuration-schema.yaml` | OpenAPI schema for the session configuration JSON document that a QR code's URL returns. The source of truth for field names and types. |
| `docs/widgets.md` | The developer guide to building a widget: the bridge, commands, events, flows, layout, testing, and platform differences. |
| `widget-starter.html` | A minimal widget template to copy for a new widget. |
| `sample_widget_controls.html` | A sample widget that exercises every control command, the stream-state event, and snapshots. |
| `PRIVACY.md` | The app's privacy policy. |

## Skills

- `widget-bridge-reference`: the exact bridge contract. Load it before writing or reviewing any widget code, and never invent message types, payload fields, or events.
- `create-widget`: scaffolds a widget from `widget-starter.html` and prints its `metadata.widgets` entry.
- `validate-session-config`: checks a session configuration against the schema and the app's runtime rules.

## Rules

- Widget code uses `window.webkit.messageHandlers.nativeApp.postMessage({ type, data })`. `window.vuplex` no longer works.
- Widget and configuration URLs in examples use `https://` and placeholder hosts such as `example.com`. Never put real tokens, internal hostnames, or private repo paths in this repo; everything here is public.
- When the bridge or schema changes, update `docs/widgets.md`, the `widget-bridge-reference` skill, and the samples together so they don't drift apart.
