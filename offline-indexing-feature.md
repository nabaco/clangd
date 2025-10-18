# Offline Background Index Creation for clangd

## Overview
This document describes the implementation of a new CLI option for clangd that enables offline creation of background index shards without starting an interactive LSP session.

## Problem Statement
Based on [issue #587](https://github.com/clangd/clangd/issues/587), users want to pre-build the clangd background index in offline scenarios (e.g., in Docker containers during build time) to avoid the indexing overhead when starting an interactive session.

## Current Behavior
- Background indexing only starts when an LSP client connects and opens a file
- The background index uses sharded storage (one shard per source file)
- No built-in way to build shards offline

## Proposed Solution
Add a `--index-project` CLI option to clangd that:
1. Reads a `compile_commands.json` file
2. Indexes all source files in the project
3. Writes background index shards to disk
4. Exits without starting an LSP server

### Command-Line Interface
```bash
clangd --index-project[=<path>] [--compile-commands-dir=<path>] [-j=<threads>]
```

### Options
- `--index-project[=<path>]`: Enable offline indexing mode. Optional path to the project root (defaults to current directory)
- `--compile-commands-dir=<path>`: Path to `compile_commands.json` (existing option, reused)
- `-j=<threads>`: Number of indexing threads (existing option, reused)
- `--background-index-priority=<priority>`: Thread priority for indexing (existing option, reused)

## Implementation Details

### Key Files to Modify
1. `clang-tools-extra/clangd/tool/ClangdMain.cpp`
   - Add new CLI option `--index-project`
   - Add logic to run indexing mode instead of LSP mode

2. `clang-tools-extra/clangd/index/Background.h`
   - Expose method to wait for indexing completion

3. `clang-tools-extra/clangd/index/Background.cpp`
   - Add functionality to index all files from compile database

### Implementation Strategy
1. Parse the new `--index-project` option
2. When enabled, skip LSP server initialization
3. Create a BackgroundIndex instance with the compilation database
4. Enqueue all files from compile_commands.json for indexing
5. Wait for indexing to complete
6. Exit cleanly

### Compatibility
- Shards created in offline mode are fully compatible with interactive sessions
- Existing background index functionality remains unchanged
- No breaking changes to existing CLI options

## Usage Examples

### Basic Usage
```bash
# Index current project (looks for compile_commands.json in current and parent dirs)
clangd --index-project

# Index with specific compile commands location
clangd --index-project --compile-commands-dir=build/

# Index with custom thread count and priority
clangd --index-project -j=8 --background-index-priority=normal
```

### Docker Container Example
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y clangd

COPY my-project /workspace
WORKDIR /workspace
RUN mkdir build && cd build && cmake ..
RUN clangd --index-project --compile-commands-dir=build/
```

## Testing Plan
1. Unit tests for CLI option parsing
2. Integration tests to verify shard creation
3. Validation tests to ensure shards are readable by interactive sessions
4. Performance tests comparing offline vs online indexing

## Benefits
1. **Faster startup**: Pre-indexed projects start immediately
2. **Container optimization**: Build index during container creation
3. **CI/CD integration**: Index as part of build pipeline
4. **Developer experience**: Consistent experience across environments

## Future Enhancements
- Support for incremental updates (only re-index changed files)
- Progress reporting with detailed statistics
- Option to verify existing shards without re-indexing
- Integration with remote index servers

## References
- Issue: https://github.com/clangd/clangd/issues/587
- Background indexing documentation: https://clangd.llvm.org/design/indexing
- LLVM project: https://github.com/llvm/llvm-project
