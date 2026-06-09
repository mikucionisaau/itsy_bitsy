# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`itsy.bitsy` is a header-only C++ library for working with bits. It provides three abstraction layers, built on top of each other:

1. **`bit_iterator<Iterator>`** — low-level iterator over bits of a single word type; the foundation everything else builds on
2. **`bit_view<Range, Bounds>`** — non-owning, potentially mutable view over bits in an arbitrary range; supports compile-time use
3. **`bit_sequence<Container>`** — owning container adaptor that wraps any sequence container (e.g., `std::vector<uint32_t>`) and exposes it as a sequence of bits

`dynamic_bitset<T>` is a convenience alias that wraps `small_bit_vector` (a small-buffer-optimized container) inside a `bit_sequence`.

The single include is `#include <itsy/bitsy.hpp>`.

## Build commands

Dependencies are fetched automatically via CMake FetchContent (requires network on first configure).

The bundled `ztd.cmake` dependency requires CMake ≥ 3.31. If the system cmake is older, install a newer one: `pip install cmake --break-system-packages`, then invoke it as `python3 -m cmake` or use the binary at `~/.local/lib/python3.12/site-packages/cmake/data/bin/cmake`.

```bash
# Configure (from repo root)
cmake -B build -DCMAKE_BUILD_TYPE=Debug -DITSY_BITSY_TESTS=ON

# Build
cmake --build build --parallel $(nproc)

# Run tests
ctest --test-dir build --output-on-failure

# Run a single test binary directly
./build/x64/Debug/bin/itsy.bitsy.tests.run_time
```

Key CMake options:
- `ITSY_BITSY_TESTS=ON` — build the Catch2 test suite
- `ITSY_BITSY_BENCHMARKS=ON` — build Google Benchmark suite
- `ITSY_BITSY_EXAMPLES=ON` — build examples
- `ITSY_BITSY_SINGLE=ON` (default) — build the single-header variant (`itsy::bitsy::single` target)

## Generating the single-header

The single-header file at `single/include/itsy/bitsy.hpp` is generated from the modular headers:

```bash
python single/single.py
```

## Code style

Formatting is enforced via `.clang-format`. Key rules: tabs for indentation (5-wide), 120-column limit, LLVM base style, `PointerAlignment: Left`, braces on their own line (GNU style).

```bash
clang-format -i include/itsy/*.hpp
```

## Architecture notes

### `bit_sequence` inherits from `bit_view`

`bit_sequence<Container>` privately inherits `bit_view<Container, word_bit_bounds<Container>>`. This means `bit_sequence` has all the read/view operations of `bit_view` plus insert/erase/push_back/resize.

### `bit_view` is the core type

`bit_view<Range, Bounds>` is parameterized on:
- **Range** — any range; mutability of the view derives from mutability of the range (e.g., `std::span<const T>` → immutable, `std::span<T>` → mutable)
- **Bounds** — controls which bits are visible: `word_bit_bounds` (all bits in all words), `bit_bounds<First, Last>` (compile-time fixed slice), `dynamic_bit_bounds` (runtime slice)

### `bit_iterator<It>` carries position within a word

Position cycles from `0` to `binary_digits_v<value_type> - 1`, then advances the underlying `It`. `.position()` and `.mask()` expose the current bit offset and its bitmask.

### Naming conventions

Internal implementation details use double-underscore prefix (`__base_t`, `__bval`). Public API is clean. Forward declarations for all types live in `include/itsy/forward.hpp`.

### Dependencies

- `ztd::idk` — type traits, `unwrap`, `to_underlying`, `to_address` utilities (fetched from github.com/soasis/idk)
- `ztd.cmake` — shared CMake prelude (fetched from github.com/soasis/cmake)
- Catch2 v2.13.7 — test framework
- Google Benchmark — benchmark framework

### Test structure

- `tests/run_time/` — Catch2 runtime tests compiled into `itsy.bitsy.tests.run_time`
- `tests/compile_time/` — tests that verify headers compile cleanly in isolation
- `tests/compile_failure/` — tests that verify invalid operations produce compile errors
- `tests/include/itsy/tests/` — shared test helpers (shared test macros, tracking allocator, constants)
