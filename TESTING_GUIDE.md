# Testing Guide for Offline Background Indexing

## Prerequisites

1. Build clangd with the patch applied
2. Have a C++ project with `compile_commands.json`

## Build Instructions

```bash
# Apply the patch to LLVM project
cd llvm-project
git apply /path/to/offline-indexing.patch

# Build clangd
mkdir build && cd build
cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_TARGETS_TO_BUILD="X86" \
      ../llvm
      
ninja clangd
```

## Test Project Setup

Create a simple test project:

```bash
mkdir test-project && cd test-project
```

Create `main.cpp`:
```cpp
#include <iostream>
#include <vector>

class Calculator {
public:
    int add(int a, int b) {
        return a + b;
    }
    
    int multiply(int a, int b) {
        return a * b;
    }
};

int main() {
    Calculator calc;
    std::cout << "5 + 3 = " << calc.add(5, 3) << std::endl;
    std::cout << "5 * 3 = " << calc.multiply(5, 3) << std::endl;
    
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    for (int n : numbers) {
        std::cout << n << " ";
    }
    std::cout << std::endl;
    
    return 0;
}
```

Create `utils.cpp`:
```cpp
#include "utils.h"
#include <algorithm>

int findMax(const std::vector<int>& numbers) {
    if (numbers.empty()) {
        return 0;
    }
    return *std::max_element(numbers.begin(), numbers.end());
}

int findMin(const std::vector<int>& numbers) {
    if (numbers.empty()) {
        return 0;
    }
    return *std::min_element(numbers.begin(), numbers.end());
}
```

Create `utils.h`:
```cpp
#ifndef UTILS_H
#define UTILS_H

#include <vector>

int findMax(const std::vector<int>& numbers);
int findMin(const std::vector<int>& numbers);

#endif // UTILS_H
```

Create `compile_commands.json`:
```json
[
  {
    "directory": "/path/to/test-project",
    "command": "/usr/bin/g++ -c -std=c++17 main.cpp -o main.o",
    "file": "main.cpp"
  },
  {
    "directory": "/path/to/test-project",
    "command": "/usr/bin/g++ -c -std=c++17 utils.cpp -o utils.o",
    "file": "utils.cpp"
  }
]
```

## Running the Test

### 1. Index the project offline

```bash
# From the test-project directory
/path/to/llvm-project/build/bin/clangd --index-project

# Or specify the path explicitly
/path/to/llvm-project/build/bin/clangd --index-project=/path/to/test-project

# With custom thread count
/path/to/llvm-project/build/bin/clangd --index-project -j=4

# With verbose logging
/path/to/llvm-project/build/bin/clangd --index-project --log=verbose
```

Expected output:
```
I[timestamp] Entering index project mode (no LSP server)
I[timestamp] Project path: /path/to/test-project
I[timestamp] Indexing project at /path/to/test-project
I[timestamp] Found 2 files to index
I[timestamp] Indexed 0/2 files
I[timestamp] Indexed 2/2 files
I[timestamp] Waiting for indexing to complete...
I[timestamp] Indexing completed successfully
```

### 2. Verify index shards were created

```bash
# Check for index shards
ls -la .cache/clangd/index/

# You should see files like:
# main.cpp.<hash>.idx
# utils.cpp.<hash>.idx
```

### 3. Test with interactive session

Start clangd normally in your editor (e.g., VS Code with clangd extension).
The symbols should be immediately available without waiting for indexing:

- Go to definition for `Calculator::add()`
- Find references for `findMax()`
- Workspace symbol search for "Calculator"

All should work immediately without indexing delay.

## Benchmark Test

### Measure indexing time

```bash
# Offline indexing
time /path/to/clangd --index-project

# Compare with interactive session startup time
# (open a file in editor and measure time until symbols are available)
```

### Expected Results

For a project with ~100 files:
- Offline indexing: ~10-30 seconds (depending on file complexity)
- Interactive first-time indexing: Similar time, but happens in background while you work
- Subsequent interactive sessions: Immediate (shards are already built)

## Docker Container Test

Create `Dockerfile`:
```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && \
    apt-get install -y build-essential cmake ninja-build git

# Copy and build clangd with patch
COPY llvm-project /workspace/llvm-project
WORKDIR /workspace/llvm-project/build
RUN cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" \
          -DCMAKE_BUILD_TYPE=Release \
          -DLLVM_TARGETS_TO_BUILD="X86" \
          ../llvm && \
    ninja clangd

# Copy test project
COPY test-project /workspace/test-project
WORKDIR /workspace/test-project

# Pre-index the project
RUN /workspace/llvm-project/build/bin/clangd --index-project

# Verify shards were created
RUN ls -la .cache/clangd/index/
```

Build and test:
```bash
docker build -t clangd-preindexed .
docker run -it clangd-preindexed /bin/bash

# Inside container, check shards exist
ls -la /workspace/test-project/.cache/clangd/index/
```

## Troubleshooting

### No files found
```
Error: No files found in compilation database
```
**Solution**: Ensure `compile_commands.json` exists and is valid.

### Index directory not created
```
Error: Failed to create index directory
```
**Solution**: Check write permissions for `.cache/clangd/index/`

### Compilation errors
If files have compilation errors, they may still be indexed but with reduced quality.
Check logs for specific error messages.

## Success Criteria

✅ ClangdMain.cpp compiles successfully
✅ clangd binary builds successfully  
✅ `--index-project` option is recognized
✅ Compilation database is loaded correctly
✅ All files from compile_commands.json are indexed
✅ Index shards are created on disk
✅ Interactive session uses pre-built shards
✅ No degradation in interactive mode performance

## Performance Expectations

| Project Size | Files | Offline Indexing Time | Startup Time Improvement |
|--------------|-------|----------------------|-------------------------|
| Small        | 1-10  | < 5 seconds          | Minimal                 |
| Medium       | 10-100| 10-60 seconds        | 5-30 seconds            |
| Large        | 100-1000 | 1-10 minutes      | 30-300 seconds          |
| Very Large   | 1000+ | 10+ minutes          | 5-30 minutes            |

## Validation Steps

1. ✅ Code compiles without errors
2. ✅ Command-line option works
3. ✅ Compilation database loading works
4. ✅ Index shards are created
5. ✅ Shards are in correct format
6. ✅ Interactive session uses shards
7. ✅ Symbols are immediately available
8. ✅ No performance regression

## Notes

- The implementation reuses existing BackgroundIndex infrastructure
- Shard format is identical to interactive mode
- No special configuration needed for interactive mode to use pre-built shards
- Index shards are automatically reused based on file digests
