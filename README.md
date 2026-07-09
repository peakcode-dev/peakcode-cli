# peakcode

AI coding agent for the terminal, written in Rust.

peakcode is a command-line coding assistant - it runs in your terminal, reads your
codebase, calls an LLM (OpenAI first, more providers later), and executes tools to edit
files, run commands, and answer questions about your project. It is modeled on
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
