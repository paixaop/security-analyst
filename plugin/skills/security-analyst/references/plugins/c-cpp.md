---
name: "C / C++ (GCC / Clang / MSVC)"
detect:
  files: ["Makefile", "CMakeLists.txt", "configure.ac", "meson.build", "Makefile.am", "SConstruct", "BUILD", "WORKSPACE", "conanfile.txt", "conanfile.py", "vcpkg.json"]
  dependencies: ["gcc", "g++", "clang", "clang++", "cmake", "make", "meson", "ninja", "conan", "vcpkg"]
  keywords: ["C", "C++", "GCC", "Clang", "MSVC", "CMake", "Makefile", "Native"]
---

## recon-agent

**C/C++ Project Discovery:**
1. Identify build system: CMake (`CMakeLists.txt`), Make (`Makefile`), Meson (`meson.build`), Autotools (`configure.ac`), Bazel (`BUILD`/`WORKSPACE`)
2. Enumerate all `.c`, `.cpp`, `.cc`, `.cxx`, `.h`, `.hpp` source files — note total count and directory structure
3. Check for compiler flags in build files: look for `-Wall`, `-Wextra`, `-Werror`, `-fstack-protector`, `-D_FORTIFY_SOURCE`, `-fPIE`, `-fPIC`, `-fsanitize=`
4. Identify third-party libraries: check `conanfile.txt`, `vcpkg.json`, `CMakeLists.txt` `find_package()` / `FetchContent`, git submodules
5. Note if AddressSanitizer, MemorySanitizer, UBSan, or ThreadSanitizer are configured in CI/test builds
6. Look for `.clang-tidy`, `.clang-format`, `compile_commands.json` — indicates static analysis tooling

## memory-safety

**C/C++ Memory Corruption — Comprehensive CWE Coverage (CWE-658/659):**

### Buffer Overflows & Out-of-Bounds Access
_(CWE-119, CWE-120, CWE-121, CWE-122, CWE-124, CWE-125, CWE-126, CWE-127, CWE-129, CWE-130, CWE-131, CWE-170, CWE-785, CWE-786, CWE-787, CWE-788, CWE-805, CWE-806)_

1. **Classic buffer overflow (CWE-120):** Grep for `strcpy`, `strcat`, `sprintf`, `gets`, `scanf("%s"` — all accept unbounded input into fixed buffers
2. **Stack-based overflow (CWE-121):** Look for fixed-size local arrays (`char buf[256]`) receiving input via `read()`, `recv()`, `fgets()` without matching size checks
3. **Heap-based overflow (CWE-122):** Check `malloc`/`calloc` allocations where subsequent writes (`memcpy`, loops) can exceed the allocated size
4. **Buffer underwrite (CWE-124):** Search for pointer arithmetic that decrements below buffer start: `ptr--` or `ptr - offset` without lower bound check
5. **Out-of-bounds read (CWE-125) / Over-read (CWE-126) / Under-read (CWE-127):** Look for reads using user-controlled indices or lengths — `buf[user_idx]`, `memcpy(dst, src, user_len)`
6. **Array index validation (CWE-129):** Check for array access with unchecked user-supplied indices — `array[atoi(input)]` without bounds validation
7. **Length parameter inconsistency (CWE-130):** Look for functions where length parameter doesn't match actual buffer size — `strncpy(dst, src, sizeof(src))` instead of `sizeof(dst)`
8. **Incorrect buffer size calculation (CWE-131):** Check for `malloc(strlen(str))` missing +1 for null terminator, or `sizeof(ptr)` instead of `sizeof(*ptr) * count`
9. **Improper null termination (CWE-170):** After `strncpy`, `read()`, `recv()` — is the buffer explicitly null-terminated? `strncpy` does NOT guarantee null termination
10. **Path manipulation buffer (CWE-785):** Check `realpath()`, `getcwd()` usage — are buffers at least `PATH_MAX` bytes?
11. **Before-start access (CWE-786) / After-end access (CWE-788):** Search for pointer arithmetic that could access before buffer start or past buffer end
12. **Out-of-bounds write (CWE-787):** Any write operation where destination offset + write size can exceed buffer allocation
13. **Incorrect length value (CWE-805) / Source buffer size (CWE-806):** `memcpy(dst, src, sizeof(src))` when `sizeof(src)` > `sizeof(dst)`, or using source length for destination bounds

### Use-After-Free, Double-Free & Lifetime Issues
_(CWE-415, CWE-416, CWE-244, CWE-562, CWE-587, CWE-761, CWE-762, CWE-763, CWE-825, CWE-910, CWE-911, CWE-1341)_

1. **Use-after-free (CWE-416):** After `free(ptr)` or `delete obj` — is the pointer used again without reassignment? Check all code paths including error handlers
2. **Double-free (CWE-415):** Can the same pointer be freed twice? Check error cleanup paths, conditional logic, and signal handlers
3. **Heap inspection (CWE-244):** Is sensitive data (passwords, keys) in heap-allocated buffers cleared with `memset_s` or `explicit_bzero` BEFORE `free()`? (Plain `memset` may be optimized away — CWE-14)
4. **Return of stack address (CWE-562):** Grep for functions returning `&local_var`, pointer to local array, or address of stack-allocated struct
5. **Fixed address assignment (CWE-587):** Search for `ptr = (type*)0xDEADBEEF` or similar hardcoded addresses — these are undefined behavior outside kernel/embedded
6. **Free not at start (CWE-761):** Is `free(ptr + offset)` called where `ptr + offset` doesn't point to start of allocation?
7. **Mismatched memory routines (CWE-762):** `malloc`/`delete`, `new`/`free`, `new[]`/`delete` (without `[]`) — mixed allocation/deallocation
8. **Invalid pointer release (CWE-763):** Freeing stack-allocated memory, freeing uninitialized pointers, freeing a pointer to a struct member
9. **Expired pointer dereference (CWE-825):** Dereferencing iterators/pointers after container modification (`std::vector` reallocation) or after scope exit
10. **Expired file descriptor (CWE-910):** Using `read()`/`write()` on a file descriptor after `close()` — the FD may have been reused
11. **Reference count errors (CWE-911):** In refcounted objects — check for missing `AddRef`/`Release`, race conditions on ref count decrement
12. **Multiple releases (CWE-1341):** Same resource (mutex, handle, fd) released more than once — check all error paths

### Integer Issues & Type Confusion
_(CWE-128, CWE-190, CWE-191, CWE-192, CWE-193, CWE-194, CWE-195, CWE-196, CWE-197, CWE-680, CWE-681, CWE-704, CWE-789, CWE-839, CWE-843, CWE-1335)_

1. **Integer overflow (CWE-190) / wraparound (CWE-128):** Check arithmetic in allocation: `malloc(n * sizeof(T))` — if `n` is user-controlled, can `n * sizeof(T)` wrap to small value?
2. **Integer underflow (CWE-191):** Subtraction that wraps unsigned integers: `size_t len = a - b` when `b > a`
3. **Integer coercion (CWE-192):** Implicit conversion between integer types of different widths — `int32_t x = (int32_t)int64_value`
4. **Off-by-one (CWE-193):** Fence-post errors in loops: `for(i=0; i<=len; i++)` instead of `i<len`, or `<= sizeof(buf)` for index
5. **Sign extension (CWE-194):** `char` promoted to `int` with sign extension — `char c = 0xFF; if(c == 0xFF)` is false on signed-char platforms
6. **Signed-to-unsigned (CWE-195):** Negative value assigned to `size_t` or `unsigned` — becomes very large positive value, bypassing length checks
7. **Unsigned-to-signed (CWE-196):** Large unsigned value assigned to signed type — becomes negative, may bypass `> 0` checks
8. **Numeric truncation (CWE-197):** 64-bit size truncated to 32-bit variable used in allocation or bounds check
9. **Integer overflow to buffer overflow (CWE-680):** Overflowed size passed to `malloc` → small allocation → large write
10. **Incorrect numeric conversion (CWE-681):** `atoi()`, `strtol()` return values used without range validation
11. **Type confusion (CWE-843):** Casting object to wrong type — especially in `void*` callback data, union type punning, C++ `reinterpret_cast` misuse
12. **Incorrect type cast (CWE-704):** Invalid `static_cast`, C-style cast between unrelated types in C++
13. **Excessive allocation (CWE-789):** User-controlled size passed directly to `malloc`/`new` without upper bound — can exhaust memory
14. **Missing minimum range check (CWE-839):** Only checking upper bound (`if (idx < max)`) but not lower bound (`idx >= 0`) for signed indices
15. **Incorrect bitwise shift (CWE-1335):** Shifting by negative amount or by >= bit-width of type — undefined behavior in C/C++

### Pointer & Memory Management
_(CWE-188, CWE-401, CWE-466, CWE-467, CWE-468, CWE-469, CWE-476, CWE-588, CWE-590, CWE-822, CWE-823, CWE-824, CWE-1325)_

1. **Memory layout reliance (CWE-188):** Code that assumes specific struct padding, endianness, or pointer size — check for `#pragma pack`, raw struct serialization
2. **Memory leak (CWE-401):** Allocated memory not freed on all code paths — especially in error-handling branches; check for early returns after `malloc`
3. **Out-of-range pointer return (CWE-466):** Functions returning pointer past buffer end or before buffer start
4. **sizeof on pointer (CWE-467):** `sizeof(ptr)` when `sizeof(*ptr)` or actual buffer size was intended — common with array-decayed-to-pointer parameters
5. **Incorrect pointer scaling (CWE-468):** Pointer arithmetic on `void*` or cast pointers where element size isn't accounted for
6. **Pointer subtraction for size (CWE-469):** `end - start` on differently-typed pointers, or using pointer diff as byte count when it's element count
7. **NULL pointer dereference (CWE-476):** Check return values of `malloc`, `calloc`, `realloc`, `fopen`, `mmap` — are they checked for NULL before use?
8. **Non-structure pointer child access (CWE-588):** Dereferencing member of cast pointer when the pointed-to memory isn't actually that struct type
9. **Free of non-heap memory (CWE-590):** Passing stack, static, or mmap'd addresses to `free()`
10. **Untrusted pointer dereference (CWE-822):** Dereferencing pointers received from untrusted sources (IPC, shared memory, deserialized data)
11. **Out-of-range pointer offset (CWE-823):** Adding user-controlled offset to pointer without bounds validation
12. **Uninitialized pointer (CWE-824):** Using a pointer variable before it's assigned — especially in complex conditional initialization
13. **Sequential memory allocation (CWE-1325):** Repeated allocations in a loop without freeing — unbounded memory growth

### Format String & Dangerous Functions
_(CWE-134, CWE-135, CWE-242, CWE-676, CWE-685)_

1. **Format string vulnerability (CWE-134):** Any `printf`-family function where format string is user-controlled: `printf(user_input)`, `syslog(LOG_ERR, user_input)`, `fprintf(f, user_input)`
2. **Multi-byte string length (CWE-135):** Using `strlen()` on multi-byte (UTF-8) strings when character count is needed — `strlen` returns byte count, not character count
3. **Inherently dangerous functions (CWE-242):** `gets()` (removed in C11), `mktemp()` — no safe usage exists, these must be replaced
4. **Potentially dangerous functions (CWE-676):** `strcpy`, `strcat`, `sprintf`, `scanf`, `strtok` (not thread-safe), `rand()` (not cryptographically secure) — verify safe alternatives are used
5. **Wrong argument count (CWE-685):** Variadic function calls (`printf`, `scanf`) where format specifiers don't match argument count — undefined behavior

### Compiler & Security Mitigation
_(CWE-14, CWE-733)_

1. **Compiler removes buffer clearing (CWE-14):** `memset(password, 0, len)` before `free()` — compiler may optimize this away as a dead store. Use `memset_s`, `explicit_bzero`, or `volatile` function pointer trick
2. **Compiler removes security code (CWE-733):** Security checks optimized away due to undefined behavior exploitation — e.g., null checks after dereference, overflow checks after the overflow

## attack-surface-http

**C/C++ HTTP Server Vulnerabilities:**
1. **Raw Socket Handling:**
   - If the project implements HTTP parsing manually (not using a library): check for header injection, request smuggling, and buffer overflows in header/body parsing
   - Look for `recv()` into fixed-size buffers without length validation
   - Check HTTP header parsing for integer overflow in `Content-Length` handling
2. **CGI / FastCGI:**
   - Are environment variables (`QUERY_STRING`, `PATH_INFO`, `HTTP_*`) used without sanitization?
   - Check for command injection via CGI parameters passed to `system()`, `popen()`, `exec*()`
3. **Embedded Web Servers (libmicrohttpd, mongoose, civetweb):**
   - Check for path traversal in static file serving
   - Verify TLS configuration and certificate validation
   - Look for missing request size limits

## attack-surface-authz

**C/C++ Authorization Patterns:**
1. **Privilege Operations:**
   - Check `setuid`, `setgid`, `seteuid` usage — is privilege dropped correctly after privileged operations?
   - Is `chroot` used without `chdir("/")` first? (CWE-243 — jail escape)
   - Look for `umask()` with chmod-style argument instead of complement (CWE-560)
2. **File Permission Handling:**
   - Check `open()`, `creat()` mode arguments — are sensitive files created with overly permissive modes?
   - Look for TOCTOU (time-of-check-time-of-use) races between `access()` and `open()` (CWE-362)
   - Is `mkstemp()` used instead of `mktemp()` for temp file creation?

## config-infrastructure

**C/C++ Build & Runtime Configuration:**
1. **Compiler Hardening Flags:**
   - `-fstack-protector-strong` or `-fstack-protector-all` (stack canaries)
   - `-D_FORTIFY_SOURCE=2` (runtime buffer overflow detection)
   - `-fPIE -pie` (position-independent executable for ASLR)
   - `-Wformat -Wformat-security` (format string warnings)
   - `-z relro -z now` (full RELRO — GOT protection)
   - `-z noexecstack` (non-executable stack)
   - Missing any of these is a finding — report which are absent
2. **Static Analysis Integration:**
   - Is `clang-tidy`, `cppcheck`, `coverity`, or `pvs-studio` configured in CI?
   - Are sanitizers (`-fsanitize=address,undefined`) enabled in test builds?
   - Is `-Werror` used to treat warnings as errors in CI?
3. **Debug Artifacts:**
   - Are debug symbols (`-g`) stripped from release builds?
   - Is `assert()` relied upon for security checks? (disabled by `-DNDEBUG` in release)
   - Look for `#ifdef DEBUG` blocks that bypass security in debug mode

## attack-surface-integrations

**C/C++ External Integration Security:**
1. **Shared Library Loading:**
   - Check for `dlopen()` with relative paths or user-controlled paths — DLL/SO hijacking
   - Is `RPATH`/`RUNPATH` set to a writable directory?
   - Look for `LD_PRELOAD` or `LD_LIBRARY_PATH` dependencies in deployment
2. **IPC & Shared Memory:**
   - Check `shm_open`, `mmap` with `MAP_SHARED` — is the shared region validated before use?
   - Look for UNIX domain sockets without permission checks
   - Verify message queue (`mq_open`, `msgget`) permissions

## logic-race-conditions

**C/C++ Concurrency Issues:**
_(CWE-362, CWE-364, CWE-366, CWE-479, CWE-543, CWE-663, CWE-689, CWE-828)_

1. **Race conditions (CWE-362):** Check for shared mutable state accessed without locks — especially global variables, static locals in multi-threaded code
2. **Signal handler races (CWE-364, CWE-828):** Signal handlers that call non-async-signal-safe functions (`malloc`, `printf`, `syslog`) — use `write()` and `sig_atomic_t` only
3. **Non-reentrant function in signal handler (CWE-479):** `strtok`, `localtime`, `getenv`, `strerror` are not safe in signal handlers
4. **Thread race (CWE-366):** Data races on shared variables without `pthread_mutex_lock` or C11 `_Atomic` / C++ `std::atomic`
5. **Non-reentrant in concurrent context (CWE-663):** Using `strtok`, `ctime`, `asctime`, `getenv` in multi-threaded code without thread-local alternatives (`strtok_r`, `localtime_r`)
6. **Singleton without sync (CWE-543):** Lazy-initialized singletons in C++ without `std::call_once` or equivalent — double-checked locking is broken without `std::atomic`
7. **Permission race on copy (CWE-689):** File created, then permissions set — window where file has default permissions

## logic-dos

**C/C++ Denial of Service Vectors:**
_(CWE-617, CWE-789, CWE-1325)_

1. **Reachable assertion (CWE-617):** `assert()` calls that can be triggered by user input — in release builds these are removed, in debug they crash
2. **Excessive allocation (CWE-789):** User-controlled values passed to `malloc`/`new`/`mmap` without upper bound limits
3. **Unbounded sequential allocation (CWE-1325):** Loops that allocate memory without freeing — can exhaust address space
4. **Stack exhaustion:** Deep recursion on user-controlled input without depth limits
5. **Algorithmic complexity:** Hash table implementations vulnerable to hash collision attacks (use SipHash or similar)

## dependency-audit

**C/C++ Dependency Concerns:**
1. **Known Vulnerable Libraries:**
   - Check versions of common C/C++ libraries: OpenSSL, libcurl, zlib, libpng, libjpeg, libxml2, sqlite
   - Cross-reference with CVE databases for known vulnerabilities
   - Look for vendored/bundled copies of libraries that may be outdated
2. **Build System Security:**
   - Check for `FetchContent` or `ExternalProject_Add` in CMake — are URLs pinned to hashes?
   - Are git submodules pinned to specific commits?
   - Check Conan/vcpkg lockfiles for dependency pinning
3. **Static vs Dynamic Linking:**
   - Statically linked vulnerable libraries won't get security updates — flag any statically linked libraries with known CVEs
   - Check for vendored source copies of libraries in the project tree

## logic-authz-escalation

**C/C++ Privilege Escalation:**
1. **SUID/SGID Binary Risks:**
   - If the binary is intended to run as SUID: check for environment variable injection (`PATH`, `LD_*`, `IFS`)
   - Verify `setuid`/`setgid` calls — is the return value checked? Failure to drop privileges is a vulnerability
   - Look for `execve` in SUID context without sanitizing the environment
2. **Capability Checks:**
   - If Linux capabilities are used: are they dropped after the privileged operation?
   - Check for `prctl(PR_SET_DUMPABLE)` — core dumps of privileged processes can leak secrets

## data-flow-tracer

**C/C++ Data Flow Patterns:**
1. **Taint Propagation:**
   - Track data from `read()`, `recv()`, `fgets()`, `scanf()`, `getenv()`, `argv[]` through the codebase
   - Check if tainted data reaches: `system()`, `popen()`, `exec*()`, `dlopen()`, SQL query construction, format strings
2. **Sensitive Data Handling:**
   - Track passwords, keys, tokens through memory — are they cleared after use?
   - Check for sensitive data written to logs, temp files, core dumps
   - Verify `mlock()` is used for pages containing cryptographic keys (prevents swapping to disk)

## git-history-injections

**C/C++ Injection Patterns to Hunt in History:**
1. **Command injection:** `system()`, `popen()` with string concatenation of user input
2. **Format string injection:** `printf(var)`, `syslog(priority, var)` where `var` was previously a string literal
3. **SQL injection in embedded databases:** String concatenation in SQLite queries (`sqlite3_exec` with `sprintf`-built query)

## git-history-data-exposure

**C/C++ Data Exposure Patterns in History:**
1. **Hardcoded credentials:** API keys, passwords, tokens in source or header files — check for commits that added then removed them
2. **Debug logging:** `fprintf(stderr, ...)` or `syslog()` calls dumping sensitive data that were later removed
3. **Core dump exposure:** Removal of `prctl(PR_SET_DUMPABLE, 0)` or addition of sensitive data to dumpable memory regions

## attack-surface-frontend

**C/C++ GUI/Desktop Application Security:**
1. **WebView Integration:**
   - If using embedded WebView (CEF, WebKitGTK, Qt WebEngine): check for JavaScript bridge injection
   - Are `file://` URLs accessible from the WebView? (local file read via web content)
   - Is the WebView loading remote content? Check for certificate validation
2. **Clipboard Handling:**
   - Is clipboard data trusted without validation? Check for injection via clipboard paste
3. **IPC with UI:**
   - If UI communicates with backend via pipes/sockets: is the IPC channel authenticated?

## logic-pipeline-exploit

**C/C++ Build Pipeline Risks:**
1. **Build Script Injection:**
   - Check `CMakeLists.txt` for `execute_process()` or `add_custom_command()` with user-controllable arguments
   - Look for shell commands in Makefiles that use environment variables without quoting
2. **Code Generation:**
   - If the build generates C/C++ source files: can the generator input be tampered to inject code?
   - Check for `#include` of generated files from untrusted build artifacts
