# peakcode-cli

Terminal client (TUI) for peakcode, the AI coding agent, written in Rust.

This is the **peakcode-cli** repository: the terminal client (binary `peakcode`). It is an
access point to the peakcode engine (`peakcode-core`); a future `peakcode-daemon` will host
long-running agent sessions so agents survive the CLI closing. It is modeled on
[opencode](https://opencode.ai).

The terminal background defaults to `none`: peakcode never forces a background color and
adapts to whatever your terminal reports.

## Status

Early development. Not yet released.

## Tech stack

- **Language:** Rust (edition 2021+)
- **First provider:** OpenAI

## Build

```bash
cargo build --release
```

## Run

```bash
cargo run
```

## Documentation

See the [`/docs`](docs/) folder.

## Security

See [`SECURITY.md`](SECURITY.md) for vulnerability reporting. peakcode is a product of
[peakssh](https://peakssh.dev).

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

MIT. See [`LICENSE`](LICENSE).
