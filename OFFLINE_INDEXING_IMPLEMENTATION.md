# Offline Background Index Implementation for clangd

This document describes the implementation of offline background indexing for clangd, addressing [issue #587](https://github.com/clangd/clangd/issues/587).

## Overview

The implementation adds a new `--index-project` CLI option to clangd that allows building background index shards without starting an interactive LSP session.

## Implementation Details

### Files Modified

1. **clang-tools-extra/clangd/tool/ClangdMain.cpp**
   - Added `--index-project` CLI option
   - Added `indexProject()` function to handle offline indexing
   - Added logic to detect and handle index project mode

### Key Features

#### 1. New CLI Option: `--index-project`
```cpp
opt<Path> IndexProject{
    "index-project",
    cat(Misc),
    desc("Index the project and write shards without starting an LSP server. "
         "Useful for pre-building background index in offline scenarios. "
         "With --index-project[=<path>], indexes all files in compile_commands.json. "
         "Path defaults to current directory if not specified."),
    init(""),
    ValueOptional,
};
```

#### 2. indexProject() Function
The core implementation:
- Loads the compilation database from the project directory
- Gets all source files from `compile_commands.json`
- Creates a `BackgroundIndex` instance
- Enqueues all files for indexing
- Waits for completion before exiting
- Provides progress logging

#### 3. Integration with Existing Infrastructure
- Uses existing `BackgroundIndex` class for indexing
- Uses existing disk-backed storage for shards
- Compatible with existing index format
- Respects existing options like `-j` (thread count) and `--background-index-priority`

## Usage

### Basic Usage
```bash
# Index current project (looks for compile_commands.json)
clangd --index-project

# Index with specific path
clangd --index-project=/path/to/project

# Index with custom thread count
clangd --index-project -j=8

# Index with custom priority
clangd --index-project --background-index-priority=normal
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

### CI/CD Integration
```yaml
# GitHub Actions example
- name: Build project
  run: cmake -B build -S .
  
- name: Pre-index with clangd
  run: clangd --index-project --compile-commands-dir=build/ -j=4
```

## Technical Implementation Details

### How It Works

1. **Compilation Database Discovery**
   ```cpp
   std::unique_ptr<tooling::CompilationDatabase> CompileDB =
       tooling::CompilationDatabase::autoDetectFromDirectory(
           ProjectPath, DatabaseLoadErrorMessage);
   ```
   The implementation uses Clang's `autoDetectFromDirectory()` to find and load `compile_commands.json`.

2. **File Enumeration**
   ```cpp
   auto AllFiles = CompileDB->getAllFiles();
   ```
   Retrieves all source files from the compilation database.

3. **Background Index Creation**
   ```cpp
   auto BgIndex = std::make_unique<BackgroundIndex>(
       TFS, CDB, std::move(IndexStorageFactory), std::move(BGOpts));
   ```
   Creates a BackgroundIndex with disk-backed storage.

4. **Indexing Execution**
   ```cpp
   BgIndex->enqueue(AllFiles);
   BgIndex->blockUntilIdleForTest(/*TimeoutSeconds=*/std::nullopt);
   ```
   Enqueues all files and waits for completion.

### Index Storage

The index shards are stored in the same location as during interactive use:
- `<project_root>/.cache/clangd/index/` for project files
- User cache directory for system headers

Each shard corresponds to one source file and contains:
- Symbol information
- References
- Relations

## Compatibility

### Forward Compatibility
- Shards created in offline mode are fully compatible with interactive sessions
- No changes to shard format or storage location
- Existing interactive sessions can use pre-built shards

### Backward Compatibility
- No changes to existing functionality
- Interactive mode unchanged
- All existing CLI options work as before

## Testing

### Manual Testing
1. Create a test project with `compile_commands.json`
2. Run: `clangd --index-project`
3. Verify shards are created in `.cache/clangd/index/`
4. Start an interactive session and verify symbols are immediately available

### Integration Testing
The implementation should be tested with:
- Small projects (< 10 files)
- Medium projects (100-1000 files)
- Large projects (10000+ files like Chromium)
- Projects with dependencies
- Projects with errors

### Performance Testing
Compare indexing time:
- Offline vs online indexing
- Different thread counts
- Different project sizes

## Error Handling

The implementation handles various error conditions:
- Missing compilation database
- Empty compilation database
- Invalid project path
- Indexing failures
- Timeout scenarios

## Future Enhancements

### Potential Improvements
1. **Incremental Updates**: Only re-index changed files
2. **Verification Mode**: Check existing shards without re-indexing
3. **Progress Bar**: More detailed progress information
4. **Statistics**: Report indexing statistics (time, size, etc.)
5. **Remote Storage**: Support for remote index storage
6. **Parallel Project Indexing**: Index multiple projects simultaneously

### Configuration Options
Future options to consider:
- `--index-only=<glob>`: Index only matching files
- `--index-exclude=<glob>`: Exclude matching files
- `--index-verify`: Verify shards after creation
- `--index-stats`: Print detailed statistics

## Comparison with clangd-indexer

| Feature | clangd-indexer | --index-project |
|---------|----------------|-----------------|
| Output Format | Static index (single file) | Background index (shards) |
| Incremental Updates | No | Compatible with interactive sessions |
| Memory Usage | High (merges all symbols) | Lower (per-file shards) |
| Use Case | Read-only scenarios | Interactive development |
| Integration | Separate tool | Built into clangd |

## References

- Original Issue: https://github.com/clangd/clangd/issues/587
- Background Index Design: https://clangd.llvm.org/design/indexing
- LLVM Project: https://github.com/llvm/llvm-project

## Patch Application

To apply this patch to the LLVM project:

```bash
cd llvm-project
git apply offline-indexing.patch
```

Then build clangd:
```bash
mkdir build && cd build
cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" ../llvm
ninja clangd
```

## Author Notes

This implementation follows the principle of minimal changes:
- Reuses existing infrastructure
- No changes to core indexing logic
- No changes to shard format
- No breaking changes to existing functionality

The implementation is designed to be easily maintainable and extensible for future enhancements.
