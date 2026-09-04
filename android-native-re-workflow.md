# Android Native Code Reverse Engineering Workflow

*Synthesized from research on native RE tooling, JNI patterns, anti-analysis bypass, and Frida native hooking. Written 2026-09-04. Target: go from "app has a suspicious .so" to understanding what it does.*

## 1. Tool list — curated

### Primary native RE platforms

**Ghidra** — NSA, free, full Android support. Auto-analysis, decompiler, scripting API (Java/Python), community ARM/ARM64 processor modules, Ghidra Bridge for remote debugging. Best free option for deep .so analysis. Import ELF, auto-detect ARM/ARM64, find JNI exports, analyze strings, trace data flow. Scriptable for batch analysis of multiple .so files.

**IDA Pro** — Hex-Rays, industry standard. Full native Android support, FLIRT/FLAIR signature matching, Hex-Rays decompiler, remote debugging via Android debugger. Expensive but the benchmark. Best for complex commercial-protected native code where signature matching and decompiler quality matter.

**JEB** — PNF Software, commercial. Android-native design, unified DEX+SO view, native code renderers, integrated JNI analysis. Good for apps where you need to see DEX and native code in the same workflow. Expensive, but purpose-built for Android.

### Supplemental static tools

**radare2** — free, command-line, powerful but steep learning curve. Good for quick inspection and scripting. Cutter is the GUI frontend worth knowing.

**angr** — binary analysis platform, constraint solving, symbolic execution. Useful for understanding branching logic and extracting constants from obfuscated native code. Steep learning curve, best for targeted questions.

**nm / objdump / readelf** — binutils, always available. `nm -D libfoo.so` for dynamic symbols, `objdump -T` for symbol table, `readelf -h` for architecture, `readelf -s` for symbols. Quick orientation before opening a full RE tool.

### Dynamic analysis

**Frida** — native hooking via `Interceptor.attach`, `Memory.scan`, `Module` APIs. The primary dynamic native analysis tool. Covered in detail in Section 5.

**GDB + Android NDK gdbserver** — classic native debugging. Push gdbserver to device, connect, set breakpoints, inspect registers/memory. Useful when Frida is detected or when you need full debugger control. NDK includes prebuilt gdbserver for target architectures.

### Quick identification

**file, unzip -l, aapt** — quick triage. `file libfoo.so` tells you architecture, `unzip -l app.apk | grep '\.so'` lists native libraries, `aapt` gives APK context.

## 2. Step-by-step native RE workflow

### Step 1 — Extract and inventory native libraries

```
unzip -l app.apk | grep '\.so$'
# or
aapt l -s app.apk | grep '\.so$'
```

List which .so files exist and their paths. Note architecture hints from paths (`lib/armeabi-v7a/`, `lib/arm64-v8a/`, `lib/x86_64/`).

```
file lib/arm64-v8a/libfoo.so
readelf -h lib/arm64-v8a/libfoo.so | grep -i 'machine\|class'
```

Confirm architecture (ARM vs ARM64 vs x86). This determines which Ghidra/IDA processor module to use.

### Step 2 — Inspect exported symbols

```
nm -D lib/arm64-v8a/libfoo.so | grep -i 'Java_'
objdump -T lib/arm64-v8a/libfoo.so | grep -i 'Java_'
readelf -s lib/arm64-v8a/libfoo.so | grep -i 'Java_'
```

Look for `Java_package_class_method` JNI exports. These are your entry points from Java to native. The naming convention makes them easy to spot.

For non-JNI native code, look for other exported symbols that might be interesting: crypto functions, network functions, file I/O, anti-analysis functions.

```
nm -D lib/arm64-v8a/libfoo.so | grep -iE 'encrypt|crypt|decrypt|aes|rsa|key|sign|verify|hash|hmac'
nm -D lib/arm64-v8a/libfoo.so | grep -iE 'socket|connect|send|recv|http|url|network|beacon|c2'
nm -D lib/arm64-v8a/libfoo.so | grep -iE 'open|read|write|stat|file|path|config|store'
```

### Step 3 — Static analysis in Ghidra (or IDA/JEB)

**Import:** File -> Import File, select the .so. Ghidra auto-detects architecture. Verify the correct processor module and language variant.

**Auto-analyze:** Run auto-analysis with default options plus:
- Enable "Decompiler Parameter Recovery" if available
- Enable "Reference" analysis
- Enable "String" analysis

**Find JNI exports:** Search for `Java_` in the symbol tree or via search. Navigate to each JNI export. The function signature will show `JNIEnv *` as first parameter, then `jclass` (static) or `jobject` (instance), then the Java-parameter equivalents.

**Analyze from JNI exports:** From a JNI export, trace:
- What Java strings/methods are referenced (GetMethodID, CallMethod, GetFieldID)
- What native functions are called (external symbols)
- What strings are used (string references, decrypted strings)
- What control flow leads to interesting operations

**String analysis:** Look at the strings window. Encrypted/obfuscated strings show up as short or non-descriptive. Cross-reference strings to see where they're used. If strings are decrypted at runtime, note the decryption function and analyze it.

**Data flow:** For functions of interest, trace arguments and return values. Follow where data comes from and where it goes. Look for:
- Data flowing from Java parameters into crypto/network/file operations
- Hardcoded keys, URLs, IPs, file paths
- Conditional logic based on device state (root, emulator, debuggable)

### Step 4 — Dynamic analysis with Frida

**Start frida-server on device:**
```
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
```

**List processes:**
```
frida-ps -U
```

**Hook a JNI export:**
```javascript
Interceptor.attach(Module.findExportByName('libfoo.so', 'Java_com_example_MyClass_nativeMethod'), {
    onEnter: function(args) {
        // args[0] = JNIEnv*, args[1] = jclass/jobject, args[2+] = Java params
        this.jenv = args[0];
        console.log('[*] Java_com_example_MyClass_nativeMethod called');
        // Log Java string params
        if (args[2] != null) {
            const str = new Java.String(args[2]);
            console.log('    param1: ' + str);
        }
    },
    onLeave: function(retval) {
        console.log('    return: ' + retval);
    }
});
```

**Hook a suspicious native function by name:**
```javascript
Interceptor.attach(Module.findExportByName('libfoo.so', 'decryptString'), {
    onEnter: function(args) {
        this.input = args[0];
        this.len = args[1];
        console.log('[*] decryptString(input=' + Memory.readUtf8String(args[0]) + ')');
    },
    onLeave: function(retval) {
        console.log('    decrypted: ' + Memory.readUtf8String(retval));
    }
});
```

**Scan memory for patterns (e.g., encrypted data markers, URLs, keys):**
```javascript
Memory.scan(Module.getBaseAddress('libfoo.so'),
           Module.getExportByName('libfoo.so', 'decryptString').sub(0x1000),
           '28 02 58 3a',
           {
               onMatch: function(addr, size) {
                   console.log('[*] Pattern found at ' + addr);
               },
               onComplete: function() {}
           });
```

**Dump decrypted strings at runtime:** Combine a hook on the decryption function with logging or writing to a file. For high-volume decryption, buffer and dump periodically rather than logging every call.

### Step 5 — Native debugging with GDB (when Frida is insufficient)

Push gdbserver and connect:
```
adb push $ANDROID_NDK/prebuilt/android-arm64/gdbserver/gdbserver /data/local/tmp/
adb shell "/data/local/tmp/gdbserver :5039 --attach <pid>"
```
On host:
```
gdb libfoo.so
target remote :5039
break Java_com_example_MyClass_nativeMethod
continue
```

Use when you need full register/memory control, when Frida is detected/blocked, or when you need to step through complex native logic instruction by instruction.

### Step 6 — Correlate and document

Cross-reference:
- Static findings (JNI exports, string references, control flow) with dynamic observations (what actually gets called, what values flow through)
- Native behavior with Java-side behavior (what triggers the native call, what does Java do with the result)
- Observed network/file/crypto activity with manifest permissions and app behavior

Document:
- JNI export table and what each does
- Key native functions and their purpose
- Network endpoints, file paths, crypto keys found
- Anti-analysis mechanisms and how they were bypassed
- Overall assessment of what the native library does

## 3. Common malware native-library patterns to recognize

### JNI bridge patterns

Typical JNI export:
```
Java_com_malware_Main_decryptString(JNIEnv *env, jobject obj, jstring input)
```

Recognize:
- `Java_com_*` naming = JNI entry point
- `JNIEnv *` first param, then `jobject` (instance) or `jclass` (static)
- Java types: `jint`, `jstring`, `jbyteArray`, `jobject`, `jclass`
- String handling: `GetStringUTFChars`, `NewStringUTF`, `GetStringLength`
- Array handling: `GetArrayLength`, `GetByteArrayElements`, `NewByteArray`, `SetByteArrayRegion`
- Object manipulation: `NewObject`, `CallMethod`, `GetFieldID`/`SetFieldID`

### String decryption patterns

Common pattern: a native function takes an encrypted string (or index) and returns decrypted string. Look for:
- A function that takes a jstring or jbyteArray and returns a jstring
- A function that takes an integer index and returns a string (string table decryption)
- Loops that XOR/decrypt data in place or into a new buffer
- Calls to `NewStringUTF` after decryption

For string table patterns, the Java side often calls a native method with an integer resource ID, and native returns the decrypted string. Find the integer-to-string mapping by hooking the function and logging all calls.

### Crypto patterns

Look for:
- OpenSSL/libtomcrypt/libgcrypt init patterns (EVP, context init, key setup)
- AES/RSA/ECC function calls
- HMAC and hash functions (SHA256, MD5)
- Key derivation (PBKDF2, custom KDF)
- Certificate/vector handling

Recognize crypto by function names (`EVP_EncryptInit`, `AES_set_encrypt_key`, etc.) and by data flow (buffers being transformed, keys being loaded, initialization vectors being set up).

### File I/O patterns

Look for:
- `open`/`openat`, `read`, `write`, `close`
- `stat`/`fstat` for file existence/size checks
- `mkdir`/`mkdirat` for directory creation
- Path strings being constructed and passed to file functions
- SharedPreferences access via JNI (often through `fopen`/`fwrite` on known paths)

Paths of interest: `/data/data/<pkg>/`, `/sdcard/`, `/data/local/tmp/`, app-specific dirs.

### Network patterns

Look for:
- `socket`/`connect`/`sendto`/`recv`/`send`/`recvfrom`
- HTTP library calls (curl, libhttp, custom HTTP)
- DNS resolution (`getaddrinfo`, `gethostbyname`)
- SSL/TLS init and handshake (OpenSSL SSL functions, custom TLS)
- Data being sent/received (buffers passed to network functions)

Recognize C2 by: periodic connect patterns, hardcoded IPs/domains, beacon-like send/receive timing, encrypted payload handling after connect.

### Anti-analysis patterns in native

Common anti-analysis functions to recognize:
- `ptrace(PTRACE_TRACEME, ...)` — self-ptrace to prevent debugging
- Reading `/proc/self/status` and checking `TracerPid`
- `syscall` wrappers around ptrace or other anti-debug syscalls
- Timing checks (`clock_gettime`, `gettimeofday` before/after operations)
- `/proc/self/maps` parsing to detect Frida/library injection
- Signal handler manipulation (`signal`, `sigaction`)
- `inotify` for file changes (detect debugger/file access)
- Network checks (`getsockname`, `getpeername`, network state checks)
- Obfuscated string comparisons (check for "frida", "gdb", "xdb", "lldb")

## 4. Anti-analysis bypass notes

### ptrace self-trace detection

Malware calls `ptrace(PTRACE_TRACEME, 0, 0, 0)` to detect if it's being debugged (only one process can ptrace-attach to a target). If it already has a tracer, ptrace fails.

Bypass:
- Hook `ptrace` to return 0 (success) regardless of arguments
- Or return -1 with errno set to disguise as "already traced" vs "no tracer"

Example:
```javascript
Interceptor.attach(Module.getExportByName(null, 'ptrace'), {
    onEnter: function(args) {
        args[0] = ptr(PTRACE_TRACEME);
    },
    onLeave: function(retval) {
        retval.replace(0); // Always succeed
    }
});
```

### TracerPid check

Malware reads `/proc/self/status` and checks the `TracerPid:` line. If non-zero, it's being debugged.

Bypass:
- Hook the read of `/proc/self/status` and return a modified version with TracerPid: 0
- Or hook the parsing function to ignore/zero the TracerPid value
- Or use Frida's built-in hiding and combine with ptrace bypass

### Frida detection via /proc/self/maps

Malware scans `/proc/self/maps` for strings like "frida", "gum", "gdb", "xdb", "lldb", or known Frida library paths.

Bypass:
- Hide Frida libraries from `/proc/self/maps` reads (Frida has some built-in support, may need customization)
- Hook the maps-reading function to filter out Frida entries
- Or patch the string comparison to never match

### Timing-based debugger detection

Malware measures time before/after an operation; abnormally slow execution indicates debugging.

Bypass:
- Hook `clock_gettime`, `gettimeofday`, `nanotime` to return controlled/fast values
- Or add a small random offset to mask timing anomalies
- Or bypass the check entirely if you can identify the specific timing validation

### Signal handler manipulation

Malware sets signal handlers and checks if they've been tampered with, or uses signals as a debugger-detection vector.

Bypass:
- Hook `signal`/`sigaction` to track and restore expected handlers
- Or leave handlers alone and work around the signal behavior
- Understand what signals the malware uses and why before patching

### General approach

Don't just patch every anti-analysis function blindly. Understand what the malware is checking and why. Sometimes the check is security theater; sometimes it guards real logic. Patch surgically and test that the behavior you care about actually runs after bypass.

## 5. Frida native hooking patterns that matter

### Basic native function hook

```javascript
const func = Module.getExportByName('libfoo.so', 'decryptString');
Interceptor.attach(func, {
    onEnter: function(args) {
        this.input = args[0];
        this.len = args[1];
    },
    onLeave: function(retval) {
        console.log('decryptString input: ' + Memory.readUtf8String(this.input));
        console.log('decryptString output: ' + Memory.readUtf8String(retval));
    }
});
```

### Replace a function entirely

```javascript
Interceptor.replace(Module.getExportByName('libfoo.so', 'isDebugged'), new NativeCallback(function() {
    return 0; // Always say not debugged
}, 'int', []));
```

Use `replace` when you want to completely override a function's behavior, not just observe it.

### Memory scanning for patterns

```javascript
const base = Module.getBaseAddress('libfoo.so');
const size = Module.getExportByName('libfoo.so', 'main').sub(base);

Memory.scan(base, size, 'ff 09 2a 4b ?? ??', {
    onMatch: function(addr, size, offset) {
        console.log('Pattern at ' + addr + ' (offset ' + offset + ')');
        // Read surrounding context
        const ctx = Memory.readByteArray(addr.add(-0x10), 0x40);
        console.log(hexdump(ctx));
    },
    onComplete: function() {
        console.log('Scan complete');
    }
});
```

Use `Memory.scan` to find code patterns, data markers, hardcoded keys, or protocol signatures without knowing exact addresses.

### Read/Write memory

```javascript
// Read string at address
const s = Memory.readCString(addr);

// Read pointer and dereference
const ptr = Memory.readPointer(addr);
const val = Memory.readU32(ptr);

// Write to memory
Memory.writeUtf8String(addr, 'new value');
Memory.writeU32(addr, 0x12345678);

// Allocate and copy
const buf = Memory.alloc(256);
Memory.copy(buf, somePointer, 256);
```

### Module introspection

```javascript
// Find base address
const base = Module.getBaseAddress('libfoo.so');

// Find export by name
const func = Module.getExportByName('libfoo.so', 'someFunction');

// Enumerate all exports
Module.enumerateExports('libfoo.so').forEach(function(exp) {
    console.log(exp.name + ' @ ' + exp.address);
});

// Enumerate symbols
Module.enumerateSymbols('libfoo.so').forEach(function(sym) {
    console.log(sym.name + ' @ ' + sym.address + ' (' + sym.type + ')');
});
```

### NativeFunction — call native functions from Frida

```javascript
const memset = new NativeFunction(Module.getExportByName(null, 'memset'), 'void', ['pointer', 'int', 'int']);
const ptr = Memory.alloc(256);
memset(ptr, 0, 256);

const memcpy = new NativeFunction(Module.getExportByName(null, 'memcpy'), 'pointer', ['pointer', 'pointer', 'int']);
memcpy(buf, src, len);
```

Use `NativeFunction` to call native library functions from your Frida script — useful for invoking decryption routines with controlled input, testing hypotheses, or reproducing behavior.

### Java + native bridge

When you need to work with Java strings or objects from a native hook:

```javascript
Java.perform(function() {
    const String = Java.use('java.lang.String');

    Interceptor.attach(Module.getExportByName('libfoo.so', 'decryptString'), {
        onEnter: function(args) {
            // Convert jstring argument to Java String
            const inputStr = String.$new(args[0]);
            this.input = inputStr.toString();
        },
        onLeave: function(retval) {
            // Convert returned jstring to Java String
            const resultStr = String.$new(retval);
            console.log('Decrypted: ' + resultStr.toString());
        }
    });
});
```

### Hooking a function by signature (when export name is obfuscated)

If the function isn't exported by a useful name, find it by scanning for a byte pattern, then hook the resolved address:

```javascript
const pattern = 'ff 09 2a 4b ?? ?? 48 8b 09';
const matches = Memory.scanSync(Module.getBaseAddress('libfoo.so'),
                                 Module.getSize('libfoo.so'),
                                 pattern);
matches.forEach(function(m) {
    const funcAddr = m.address;
    Interceptor.attach(funcAddr, {
        onEnter: function(args) { console.log('Hit at ' + funcAddr); },
        onLeave: function(retval) {}
    });
});
```

### Frida detection bypass patterns

```javascript
// Hide from /proc/self/maps scanning
// (Frida's built-in stealth helps, but some malware reads maps directly)
const open = Module.getExportByName(null, 'open');
Interceptor.attach(open, {
    onEnter: function(args) {
        const path = Memory.readUtf8String(args[0]);
        if (path.indexOf('maps') >= 0 && path.indexOf('/proc/self/') >= 0) {
            // Return a fake fd or modify behavior
            console.log('[*] Maps access detected');
        }
    }
});
```

Combine multiple bypass hooks: ptrace + TracerPid + maps scanning + timing + signal handlers. Test after each addition to make sure you haven't broken the behavior you're trying to observe.

## Bottom line

Native RE workflow: extract .so → identify architecture → enumerate JNI exports and suspicious symbols → static analyze in Ghidra/IDA/JEB → dynamic hook with Frida → debug with GDB if needed → correlate and document.

Recognize patterns: JNI naming, string decryption, crypto init, file/network I/O, anti-analysis (ptrace, TracerPid, maps, timing, signals).

Bypass surgically: hook the specific anti-analysis function, understand what it checks, test that real behavior still runs.

Frida native hooking is the workhorse: Interceptor.attach/replace, Memory.scan, Module introspection, NativeFunction for calling back into native code, Java+native bridge for string handling.

Ghidra is the best free static platform. IDA Pro is the benchmark if you have it. JEB is purpose-built for Android DEX+SO work. radare2/cutter and angr are supplementary.
