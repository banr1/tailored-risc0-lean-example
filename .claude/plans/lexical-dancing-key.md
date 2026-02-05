# Plan: Create CLAUDE.md

## Task
Create a CLAUDE.md file for the tailored-risc0-lean-example repository.

## Content to include

### Build/Run Commands
- `just build` — full build pipeline (Lean → C IR → CMake → Cargo)
- `just clean` — clean all build artifacts
- `target/release/host N` — run with numeric input
- Individual steps: `cd guest && lake build`, `cd guest_build && just build`, `cargo build --release`

### Required Environment Variables
- `LEAN_RISC0_PATH` — path to Lean RISC0 runtime (~/.lean-risc0)
- `RISC0_TOOLCHAIN_PATH` — path to RISC0 toolchain

### Architecture Overview
- Multi-language project: Lean 4 → C → Rust via RISC Zero zkVM
- Build pipeline: Lake compiles Lean to C IR → CMake cross-compiles to RISC-V static lib → Cargo links everything into guest ELF → Host proves execution
- Key directories and their roles: guest/, guest_build/, methods/, host/
- FFI boundary: Lean `@[export risc0_main]` → C `lean_risc0_main()` → Rust safe wrapper
- Data marshaling: ByteArray (Lean) ↔ C buffer ↔ Vec<u8> (Rust) ↔ u32 (Host I/O)

### Custom Lean Program Instructions
- Replace `guest/` directory with your Lean 4 project
- Must have `Guest.lean` with `@[export risc0_main] def risc0_main (input : ByteArray) : ByteArray`
- Lean version pinned to 4.22.0

## File to create
`/Users/banriyanahama/workspace/tailored-risc0-lean-example/CLAUDE.md`

## Verification
- Confirm the file is well-structured and concise
- Ensure no duplication or generic advice
