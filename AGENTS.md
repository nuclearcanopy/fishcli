# fishcli - instructions for OpenCode

Standing instructions for this project. Read before working.

## Workflow

- Progressive commits only. No monolithic dumps. One small, functional change per commit.
- Both remotes (Codeberg `origin`, GitHub `github`) must stay in sync. Ask before pushing if a remote is missing.
- Never run `git config` in this repo. Git identity is managed declaratively in `~/nix-config`.

## Game design

- Stardew-style fishing minigame: vertical bar, player-controlled rectangle, fish moves inside, keep fish in rectangle to fill catch meter. Rectangle shrinks and fish gets faster with difficulty.
- Pseudo-graphics in the Dwarf-Fortress vein.
- Overworld is a tile map with proper houses built from `#` characters (walls). Characters (player, NPCs) are single `@` glyphs.
- Progression: rod shop (hundreds of rod upgrades) plus fishing school (technique upgrades, e.g. permanently +1 to rectangle height, faster bites, etc.).

## Stack

- Rust, ratatui + crossterm.
- Declarative dev shell via `flake.nix`.

## Save-file evolution (forward-compatible)

This is a continuously-updating game. Older saves must keep loading after new versions add fields. The convention is non-negotiable; players should never need to restart on an update.

- Every new field on `SaveData` (or on any struct nested inside it - `Stats`, `Skills`, `OwnedRods`, `BaitStock`, `EquippedTackle`, etc.) gets `#[serde(default)]`. For non-`Default` types use `#[serde(default = "fn_name")]` with an explicit default function.
- Never rename or remove an existing field without a wire-version bump. Rename = old saves silently lose data; removal = same. If a field becomes obsolete, leave it on the struct (mark unused with a leading underscore) and stop reading from it.
- When adding a new dimension, biome, item, or quest, the `caught`/`mastery`/etc. parallel vectors are length-checked and zero-extended on load - do not reorder existing entries in `assets/*.json`, only append.
- Breaking wire-layout changes (new header field, new crypto envelope) bump the version byte at `save.rs:246`-ish and add a v1-to-v2 conversion in `decrypt_opaque`. The loader already rejects unknown versions cleanly, so a partial rollout does not corrupt files.
- There is a reminder comment block above the `SaveData` struct in `src/save.rs`; keep it in sync if the rule evolves.

## Building

- The user iterates from this CLI, so build speed matters.
- Always run `nix develop` first if the shell does not already have `cargo`/`rustc` on `PATH`. The flake's dev shell provides cargo, rustc, rustfmt, clippy, rust-analyzer, mold, sccache, and clang. `sccache` is auto-configured by the flake's `shellHook` and caches rustc artifacts to `~/.cache/sccache`.
- Default `cargo build --release` is ~3s incrementally (LTO off, codegen-units=16, incremental on).
- Never use `--profile ship` unless explicitly asked to make a distributable binary - that re-enables full LTO and takes ~2min.
- Use `cargo check` for a compile sanity check; `cargo build` (debug) is also fast and runs the game fine for testing UI.
- The linker block in `.cargo/config.toml` is commented out by default (needs `mold` + `clang` on `PATH`; the flake's dev shell provides them). Uncomment only after `nix develop`.
