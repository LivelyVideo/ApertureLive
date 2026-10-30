# ApertureLive

Developer resources for integrating with Aperture Live, the live-streaming app for iOS and Android.

- [Session configuration schema](session-configuration-schema.yaml): the JSON document a session QR code points to.
- [Building a widget](docs/widgets.md): add your own web controls to the broadcast screen and drive the app over its JavaScript bridge.
- [Widget starter template](widget-starter.html): a minimal page to copy.
- [Sample controls widget](sample_widget_controls.html): a button for every control command, plus stream state and snapshots.
- [Privacy policy](PRIVACY.md)

## Using Claude Code

This repo includes [Claude Code](https://claude.com/claude-code) skills. Open the repo in Claude Code and ask it to create a widget (the `create-widget` skill) or to check a session configuration (the `validate-session-config` skill). The skills use the bridge contract in `docs/widgets.md`.
