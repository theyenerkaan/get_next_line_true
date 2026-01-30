# get_next_line

A robust file reading utility function in C that returns a single line from a file descriptor on each call. This project implements efficient buffered I/O with static memory management, demonstrating low-level file handling and string manipulation without standard library dependencies.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Technical Details](#technical-details)
- [Limitations & Notes](#limitations--notes)
- [License](#license)

## Overview

`get_next_line` is a fundamental systems programming project that solves the classic problem of reading text files line-by-line in C. Unlike standard library functions like `fgets()`, this implementation provides complete control over buffer management, memory allocation, and file descriptor handling.

**Problem solved**: Efficiently read text files of arbitrary size, one line at a time, while handling multiple file descriptors simultaneously and managing memory safely.

**Why it exists**: Part of the 42 School curriculum, this project teaches critical systems programming concepts including:
- Low-level file I/O with `read()` system call
- Static variable lifetime and persistence
- Dynamic memory management
- Buffer overflow prevention
- Edge case handling in string operations

**High-level behavior**: Each call to `get_next_line(fd)` returns the next line from the file descriptor, including the newline character. The function maintains internal state using static variables to preserve unprocessed data between calls.

## Features

### Core Functionality
- **Line-by-line reading**: Returns one complete line per function call
- **Newline preservation**: Includes `\n` character in returned string (if present)
- **Arbitrary file sizes**: Handles files of any length efficiently
- **Configurable buffer**: `BUFFER_SIZE` can be set at compile time
- **Memory efficient**: Only allocates memory for current line and buffer
- **EOF handling**: Returns `NULL` when end of file is reached
- **Error resilience**: Handles read errors gracefully with proper cleanup

### Bonus Features
- **Multiple file descriptors**: Simultaneous reading from up to 10,240 different files
- **File descriptor array**: Uses static array to track state per descriptor
- **Concurrent reads**: Can interleave reads from multiple files without losing position

### Input Validation
- Invalid file descriptors (fd < 0)
- Invalid buffer sizes (BUFFER_SIZE <= 0)
- Out-of-range file descriptors (fd > 10240 in bonus)
- Read system call failures
- Empty files or empty lines

## Project Structure

```
get_next_line_true/
├── get_next_line.h              # Main header with function prototypes
├── get_next_line.c              # Core implementation
├── get_next_line_utils.c        # Helper functions (strlen, strchr, strjoin)
├── get_next_line_bonus.h        # Bonus header for multiple FDs
├── get_next_line_bonus.c        # Enhanced implementation for multiple FDs
├── get_next_line_utils_bonus.c  # Bonus utility functions
└── Makefile                     # Build configuration
```

### Key Files

**get_next_line.c**
- `get_next_line(int fd)` - Main function, returns next line from file descriptor
- `ft_read_file()` - Reads from file into buffer until newline or EOF
- `ft_get_line()` - Extracts complete line from accumulated buffer
- `remove_line()` - Updates static buffer, removing processed line

**get_next_line_utils.c**
- `ft_strlen_gnl()` - Calculate string length (NULL-safe)
- `ft_strchr_gnl()` - Check if character exists in string
- `ft_strjoin_gnl()` - Concatenate two strings with memory reallocation

**get_next_line_bonus.c**
- Multiple file descriptor support via static array `garbage[10240]`
- Same core logic with per-FD state management

## Installation

### Prerequisites

- **Compiler**: GCC or Clang
- **OS**: UNIX-like system (Linux, macOS, BSD)
- **Standards**: C99 or later

### Building

```bash
# Clone the repository
git clone <repository-url>
cd get_next_line_true

# Compile as static library
make

# Clean build artifacts
make clean

# Full clean (removes library)
make fclean

# Rebuild from scratch
make re
```

### Compilation Flags

```makefile
CFLAGS = -Wall -Wextra -Werror
```

- **-Wall**: Enable all standard warnings
- **-Wextra**: Enable extra warnings
- **-Werror**: Treat warnings as errors

### Custom Buffer Size

Compile with custom buffer size using `-D` flag:

```bash
gcc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line.c get_next_line_utils.c
```

## Usage

### Basic Integration

```c
#include "get_next_line.h"
#include <fcntl.h>  // for open()
#include <stdio.h>  // for printf()

int main(void)
{
    int     fd;
    char    *line;

    // Open file for reading
    fd = open("example.txt", O_RDONLY);
    if (fd == -1)
        return (1);

    // Read file line by line
    while ((line = get_next_line(fd)) != NULL)
    {
        printf("%s", line);
        free(line);  // Always free returned line
    }

    close(fd);
    return (0);
}
```

### Multiple File Descriptors (Bonus)

```c
#include "get_next_line_bonus.h"
#include <fcntl.h>

int main(void)
{
    int     fd1, fd2;
    char    *line1, *line2;

    fd1 = open("file1.txt", O_RDONLY);
    fd2 = open("file2.txt", O_RDONLY);

    // Interleaved reading from two files
    line1 = get_next_line(fd1);
    line2 = get_next_line(fd2);
    line1 = get_next_line(fd1);
    line2 = get_next_line(fd2);

    // Process lines...
    free(line1);
    free(line2);

    close(fd1);
    close(fd2);
    return (0);
}
```

### Reading from Standard Input

```c
#include "get_next_line.h"
#include <unistd.h>  // STDIN_FILENO

int main(void)
{
    char *line;

    while ((line = get_next_line(STDIN_FILENO)) != NULL)
    {
        // Process input line
        free(line);
    }
    return (0);
}
```

### Expected Output

**Input file (example.txt):**
```
Hello, World!
This is line 2
Final line without newline
```

**Output:**
```
"Hello, World!\n"
"This is line 2\n"
"Final line without newline"
```

## Technical Details

### Algorithm & Data Flow

#### State Management with Static Variables

```c
char *get_next_line(int fd)
{
    static char *garbage;  // Persists between function calls
    // ...
}
```

**Static variable behavior:**
- Allocated once, persists across function calls
- Stores leftover data from previous reads
- Automatically initialized to NULL on first call
- Must be freed when file is exhausted

#### Buffer Reading Strategy

1. **Allocate temporary buffer** of `BUFFER_SIZE` bytes
2. **Read loop**: Continue reading until newline found or EOF
3. **Accumulate data**: Concatenate chunks into persistent buffer
4. **Extract line**: Copy from start to newline (inclusive)
5. **Update state**: Remove processed line from static buffer
6. **Return**: Dynamically allocated line string

**Pseudocode:**
```
loop:
    if (buffer contains '\n' OR end of file):
        extract line up to '\n'
        save remainder in static buffer
        return line
    else:
        read(fd, temp_buffer, BUFFER_SIZE)
        append temp_buffer to static buffer
        continue loop
```

### Memory Management

**Allocation Strategy:**
- **Dynamic allocation**: All strings allocated with `malloc()`
- **Caller responsibility**: User must `free()` returned line
- **Internal cleanup**: Static buffer freed on EOF or error
- **No leaks**: Every allocated byte is tracked and freed

**Memory Safety Features:**
- NULL pointer checks before dereferencing
- Allocation failure handling
- Proper cleanup on error paths
- Double-free prevention with NULL assignment

**Critical Pattern:**
```c
// Always check allocation success
line = malloc(sizeof(char) * (size + 1));
if (!line)
    return (NULL);

// Free old buffer when creating new one
free(old_buffer);
old_buffer = new_buffer;
```

### System Calls

**read() - Core I/O Operation**
```c
ssize_t read(int fd, void *buf, size_t count);
```

- **fd**: File descriptor (0=stdin, 1=stdout, 2=stderr, 3+=files)
- **buf**: Destination buffer for read data
- **count**: Maximum bytes to read (BUFFER_SIZE)
- **Returns**: Bytes read, 0 on EOF, -1 on error

**Error Handling:**
```c
i = read(fd, temp, BUFFER_SIZE);
if (i == -1) {
    // Read failed (invalid FD, I/O error, etc.)
    cleanup_and_return_null();
}
if (i == 0 && !ft_strlen(garbage)) {
    // EOF reached with no pending data
    return (NULL);
}
```

### Buffer Size Optimization

**Trade-offs:**

| Buffer Size | Pros | Cons |
|-------------|------|------|
| Small (1-32) | Less memory usage | More system calls (slower) |
| Medium (64-1024) | Balanced performance | Moderate memory |
| Large (4096+) | Fewer system calls | Higher memory footprint |

**Recommended:** 4096 bytes (typical page size on UNIX systems)

**42 Project Default:** Varies (often tested with edge cases like 1, 9999999)

### Edge Cases Handled

1. **Empty file**: Returns NULL immediately
2. **File without trailing newline**: Returns last line without `\n`
3. **Single character reads** (BUFFER_SIZE=1): Still works correctly
4. **Extremely long lines**: No arbitrary line length limit
5. **Binary files**: Handles any byte values (including `\0`)
6. **Multiple consecutive newlines**: Each returned as separate line `"\n"`
7. **File descriptor reuse**: Properly resets state between files

### Performance Characteristics

**Time Complexity:**
- Best case: O(n) where n = line length
- Worst case: O(n) - linear scan of input
- Each character read exactly once from file

**Space Complexity:**
- O(n + b) where n = line length, b = BUFFER_SIZE
- Static buffer grows with leftover data
- Released when file reading completes

**System Call Efficiency:**
- Calls `read()` only when buffer exhausted
- Minimizes context switches to kernel
- Amortized O(1) reads per line for typical files

### Security Considerations

**Buffer Overflow Prevention:**
- No fixed-size buffers (all dynamic allocation)
- Explicit bounds checking in loops
- NULL termination guaranteed

**Integer Overflow Protection:**
- Size calculations use `size_t` (unsigned)
- Allocation sizes validated before `malloc()`

**Resource Exhaustion:**
- No infinite loops (all loops have exit conditions)
- Bounded by file size and available memory
- Proper cleanup on allocation failures

## Limitations & Notes

### Known Limitations

1. **Maximum file descriptors (bonus)**: Limited to 10,240 simultaneous FDs
   - System limit often lower (`ulimit -n`)
   - Array size fixed at compile time

2. **Memory usage**: O(n) where n = longest unprocessed line
   - Long lines without newlines consume memory
   - Static buffer persists until file exhausted

3. **Not thread-safe**: Static variables make function non-reentrant
   - Cannot safely call from multiple threads on same FD
   - Would require mutex locking or thread-local storage

4. **Binary file support**: Limited to text processing
   - Stops at first `\0` byte in string functions
   - Not suitable for binary protocols

5. **No seek operations**: Sequential reading only
   - Cannot go backwards or skip lines
   - File position controlled by `read()` calls

### Edge Cases

**Handled correctly:**
- Files ending without `\n`
- Empty files (returns NULL)
- Single-byte buffer sizes
- Very large buffer sizes
- Multiple consecutive newlines

**Not supported:**
- Wide characters / UTF-16/UTF-32
- Line length limits (unlimited)
- Non-blocking file descriptors (may behave unexpectedly)

### Design Decisions

**Why static variables?**
- Persist state between function calls
- Avoid global variables (better encapsulation)
- Standard approach for stateful single-call functions

**Why include newline in return value?**
- Preserves original file structure
- Allows distinction between line endings
- Consistent with POSIX conventions

**Why dynamic buffer?**
- Support arbitrarily long lines
- Efficient memory usage (only what's needed)
- Avoid buffer overflow vulnerabilities

### 42 School Constraints

- **Forbidden functions**: Only `read()`, `malloc()`, `free()` allowed
- **No libft**: Must implement own string utilities
- **Norm compliance**: Strict coding standards (25 lines per function, etc.)
- **No memory leaks**: Must pass Valgrind with zero leaks
- **No global variables**: Allowed only static variables

## Compiler Flags for Testing

```bash
# With Valgrind memory checking
gcc -Wall -Wextra -Werror -D BUFFER_SIZE=42 -g \
    get_next_line.c get_next_line_utils.c main.c -o gnl_test

valgrind --leak-check=full --show-leak-kinds=all ./gnl_test

# With address sanitizer
gcc -Wall -Wextra -Werror -D BUFFER_SIZE=1 -fsanitize=address \
    get_next_line.c get_next_line_utils.c main.c -o gnl_test
```

## License

This project is for educational purposes as part of the 42 School curriculum. The implementation adheres to strict academic integrity standards and project constraints.

---

**Technical Highlights for Recruiters:**

This project demonstrates proficiency in:
- **Low-level I/O**: Direct use of POSIX `read()` system call
- **Memory management**: Manual allocation/deallocation without leaks
- **State machines**: Stateful function design with persistent data
- **Buffer management**: Efficient chunked reading strategies
- **Error handling**: Robust validation and graceful failure modes
- **String manipulation**: Custom implementations without libc dependencies
- **Systems programming**: Understanding of file descriptors, EOF, and Unix I/O model

Relevant for roles in: **Systems Programming, Embedded Systems, Operating Systems Development, Low-Level Libraries, File System Engineering, Kernel Development**
