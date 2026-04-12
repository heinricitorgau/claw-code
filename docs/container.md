# Container workflows — claw-code

## Context

The Rust runtime already included container detection before this document was added:

- `rust/crates/runtime/src/sandbox.rs` detects Docker/Podman/container markers: `/.dockerenv`, `/run/.containerenv`, matching env vars, and `/proc/1/cgroup` hints.
- `rust/crates/rusty-claude-cli/src/main.rs` exposes that state through `claw sandbox` / `cargo run -p rusty-claude-cli -- sandbox`.
- `.github/workflows/rust-ci.yml` runs on `ubuntu-latest` without a Docker or Podman container job.

The root [`../Containerfile`](../Containerfile) adds a canonical container image for Docker and Podman users. It does not copy the repository into the image; the recommended flow is to bind-mount your checkout into `/workspace` so edits stay on the host.

**Rationale for bind-mount approach**: avoids container-owned `target/` artifacts appearing in the host checkout; keeps the image reusable across checkout states without rebuilding.

## Build the image

### Docker

```bash
docker build -t claw-code-dev -f Containerfile .
```

### Podman

```bash
podman build -t claw-code-dev -f Containerfile .
```

## Run `cargo test --workspace` in the container

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev \
  cargo test --workspace
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev \
  cargo test --workspace
```

For a fully clean rebuild, prepend `cargo clean &&` before `cargo test --workspace`.

## Open a shell in the container

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

Inside the shell:

```bash
cargo build --workspace
cargo test --workspace
cargo run -p rusty-claude-cli -- --help
cargo run -p rusty-claude-cli -- sandbox
```

The `sandbox` command is a useful verification check: inside Docker or Podman it should report `In container true` and list the markers the runtime detected.

## Bind-mount a second repository alongside claw-code

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -v "$HOME/src/other-repo":/repo \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -v "$HOME/src/other-repo":/repo:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

Then, for example:

```bash
cargo run -p rusty-claude-cli -- prompt "summarize /repo"
```

## Notes

| Concern | Detail |
|---------|--------|
| Docker vs Podman | Both use the same checked-in `Containerfile` |
| `:Z` suffix in Podman examples | Required for SELinux relabeling on Fedora/RHEL-class hosts |
| `CARGO_TARGET_DIR=/tmp/claw-target` | Avoids leaving container-owned `target/` artifacts in the bind-mounted checkout |
| Non-container local development | Use [`../USAGE.md`](../USAGE.md) and [`../rust/README.md`](../rust/README.md) |
