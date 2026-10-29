# Repository Guidelines

Check the root `CLAUDE.md` file for repo defaults.
Consult crate-specific `CLAUDE.md` or `AGENTS.md` files when changing files in those crates.

## Cursor Cloud specific instructions

The base image points `cc` and `c++` at clang. RocksDB's C++ build then passes `--target` to clang and fails with `fatal error: 'limits' file not found`. Before compiling, select GCC:

```bash
sudo update-alternatives --set cc /usr/bin/gcc
sudo update-alternatives --set c++ /usr/bin/g++
```

`postgres`, `initdb`, `pg_ctl`, and `pg_isready` must be on `PATH`. Ubuntu installs them under `/usr/lib/postgresql/*/bin`. `sui-pg-db` tests start a temporary server with those commands and do not use a system database on port 5432.

The Rust toolchain is pinned by `rust-toolchain.toml`. Compiling also needs `protoc`, `libssl-dev`, `libclang-dev`, and `libpq-dev`. Library tests use `SUI_SKIP_SIMTESTS=1 cargo nextest run -p <crate>`. Simulator tests use `cargo simtest`. On a small VM, set `CARGO_BUILD_JOBS=3`.

A local network with a faucet:

```bash
mkdir -p /tmp/sui-localnet
sui genesis -f --with-faucet --working-dir /tmp/sui-localnet
SUI_CONFIG_DIR=/tmp/sui-localnet sui start --with-faucet --network.config /tmp/sui-localnet
```

The fullnode listens on `http://127.0.0.1:9000`. Request coins from `http://127.0.0.1:9123/gas`.
