# LWJGL 3.2.2 → ppc64 BE (big-endian) Linux Port

## Target
- **Arch**: ppc64 (big-endian), ELFv2, 64kb pagesize
- **OS**: Arch PowerPC Linux, kernel 7.1.1
- **Toolchain**: `powerpc64-linux-gnu-gcc` (cross from x86_64) or native gcc on ppc64

## Build system
- Ant-based (`build.xml` + platform overrides in `config/{linux,macos,windows}/`)
- Native compilation in `config/linux/build.xml` using `<build>` / `<build_simple>` macros
- Build arch is hardcoded to `x86`/`x64` via `build-definitions.xml:77-80`

## Patches needed

### 1. Build system (`config/build-definitions.xml`)
- Add `ppc64` as valid `build.arch` value (currently only `x86`/`x64`)
- Must handle detection: `os.arch` on ppc64 BE is `ppc64` or `powerpc`

### 2. Linux native build (`config/linux/build.xml`)
- Replace `-m64`/`-m32` with `-mcpu=powerpc64 -mbig-endian` for ppc64
- Remove `-mfpmath=sse -msse -msse2` (x86-specific)
- Remove `--wrap memcpy` on ppc64 (x86-specific glibc workaround)
- Set `LIB_POSTFIX` to `""` (ppc64 is always 64-bit)
- Add `-DLWJGL_PPC64` define
- Set gcc suffix via env var (e.g. `powerpc64-linux-gnu-`)
- Disable modules that require x86-specific code (SSE, meow — uses AES/SSE4.2; tootle — uses SSE via RayTracer)

### 4. LMDB (`modules/lwjgl/lmdb/src/main/c/mdb.c`)
- Has own endian detection via `BYTE_ORDER`/`_BIG_ENDIAN` — should work
- Line 250-252: `MISALIGNED_OK` is only set for x86/x64. On ppc64, unaligned access will trap. Either add `#ifndef __powerpc64__` guard or ensure all access is aligned.

### 5. LZ4 (`modules/lwjgl/lz4/src/main/c/`)
- Line 138-158: PPC64LE-specific `-O2` workaround — not relevant for BE
- Endian detection in `lz4.c:221` uses runtime union check — works on all archs
- Compression/decompression has BE/LE paths — should work

### 6. stb_vorbis (`modules/lwjgl/stb/src/main/c/stb_vorbis.c`)
- Line 538-544: `STB_VORBIS_BIG_ENDIAN` defaults to 0 — **must define to 1** for BE target
- Can pass `-DSTB_VORBIS_BIG_ENDIAN=1` in compiler flags

### 7. tinyexr / miniz (`modules/lwjgl/tinyexr/src/main/c/tinyexr.h`)
- Uses `MINIZ_LITTLE_ENDIAN` macro — verify it detects BE correctly
- Has incomplete BE support in PIZ wavelet transform (marked with `@todo`)

### 8. xxHash (`modules/lwjgl/xxhash/src/main/c/xxhash.c`)
- Uses `__attribute__((packed))` unions for unaligned reads — should work
- Uses `XXH_alignment` runtime check — should work

### 9. zstd (`modules/lwjgl/zstd/src/main/c/`)
- Uses `MEM_isLittleEndian()` runtime check and packed structs — should work

### 10. libdivide (if used)
- Check for x86 BMI2/AVX2 dependencies

### 11. SSE module
- **Must be disabled** — x86-specific intrinsics (`-msse3`)

### 12. meow hash module
- **Must be disabled** — uses AES-NI / SSE4.2 (`-maes -msse4.2`)

### 13. Java side (`MemoryUtil.java`)
- `NATIVE_ORDER = ByteOrder.nativeOrder()` — works automatically on BE
- `NativeShift` interface — has BE implementation, should be fine
- **Cannot change `NATIVE_ORDER` globally** — used for UTF16 charset selection and NativeShift access (would break memory I/O)

### 14. Pixel data endianness — CRITICAL for Minecraft
- Minecraft's `NativeImage` stores pixels as packed ints using `getInt()`/`putInt()`
- Component extraction: `getRed(int)` → `color & 0xFF`, `getBlue` → `(color >> 16) & 0xFF`, etc.
- Format expected: `0xAABBGGRR` (Alpha MSB, Blue, Green, Red LSB)
- stb_image returns raw bytes `[R, G, B, A]` in native memory

**On LE (x86_64):** `getInt()` reads `[R, G, B, A]` as LE int = `0xAABBGGRR` ✓
**On BE (ppc64):** `getInt()` reads `[R, G, B, A]` as BE int = `0xRRGGBBAA` ✗ (components swapped!)
  - Purple (0xFF00FF) → Yellow (0xFFFF00), Black → Red — matches reported bug

**Fix:**
1. `STBImage.java`: `pixelBufferSafe()` wrapper returns ByteBuffers with `LITTLE_ENDIAN` order
2. Minecraft's `NativeImage` constructor calls `.order(ByteOrder.nativeOrder())` → **overrides fix back to BE**
3. Must modify the Minecraft jar: change the call to `ByteOrder.LITTLE_ENDIAN` instead

## Cross-compile workflow (from x86_64)

```bash
# Install cross-toolchain
sudo pacman -S powerpc64-linux-gnu-gcc       # Arch
# or: apt install gcc-powerpc64-linux-gnu    # Debian/Ubuntu

# Build dyncall for ppc64
git clone https://github.com/LWJGL-CI/dyncall.git
cd dyncall
./configure --target=ppc64-linux-gnu
make
# Copy libdyncall_s.a libdyncallback_s.a libdynload_s.a to lib/linux/ppc64/

# Build LWJGL
LWJGL_BUILD_ARCH=ppc64 \
LWJGL_BUILD_OFFLINE=true \
ant compile-native-platform
```

## Direct compile on ppc64 machine

```bash
# On the ppc64 box:
# 1. Ensure Java 8+ JDK and Ant are installed
# 2. Build dyncall natively (see above)
# 3. Place dyncall static libs at lib/linux/ppc64/
# 4. Build native:

cd lwjgl3-3.2.2
# GLFW, OpenAL, etc. are Java-only bindings — system .so loaded at runtime
# Only SSE, Meow, and Tootle need disabling (x86 intrinsics):
ant -Dbuild.arch=ppc64 \
  -Dbinding.sse=false -Dbinding.meow=false -Dbinding.tootle=false \
  -Dbuild.offline=true \
  compile-native-platform
```

## Third-party native deps
All downloaded from `https://build.lwjgl.org/` — no ppc64 builds exist.
Must build from source for these:
- dyncall (essential — function call bridge)
- GLFW (windowing)
- OpenAL (audio)
- Assimp (model loading, optional)
- bullet (physics, optional)
- jemalloc (allocator, optional)
- GLFW (windowing — Java-only binding, system lib loaded at runtime)
- OpenAL (audio — Java-only binding, system lib loaded at runtime)
- stb (images/fonts/audio — bundled as C sources, compiled with LWJGL)
- lz4, zstd, xxhash (bundled as C sources)

## Verification
After building `liblwjgl.so`, test with:
```bash
file liblwjgl.so    # should show "ELF 64-bit MSB executable, 64-bit PowerPC ..."
readelf -h liblwjgl.so | grep -i machine   # EM_PPC64
```

## Minecraft 1.16.5 — Patching NativeImage endianness

### Problem
stb_image returns pixel data as raw bytes `[R, G, B, A]` in native memory. On ppc64 BE,
`getInt()` reads these as `0xRRGGBBAA`, but Minecraft expects `0xAABBGGRR` (LE format).
This causes all texture colors to have swapped channels (purple→yellow, black→red).

### Fix 1: Minecraft's NativeImage (may be required)
Minecraft's `NativeImage` constructor calls `.order(ByteOrder.nativeOrder())`
on the pixel buffer, overriding LWJGL's LITTLE_ENDIAN setting back to BIG_ENDIAN.
The Minecraft jar must be patched to use `ByteOrder.LITTLE_ENDIAN` instead.

**Locate NativeImage** in the Minecraft jar:
```bash
cd ~/.minecraft/versions/1.16.5/
mkdir -p /tmp/mc_patch && cd /tmp/mc_patch
jar xf 1.16.5.jar
grep -l "nativeOrder\|NativeImage" $(find . -name "*.class") | head -5
```

**On vanilla 1.16.5:** NativeImage is obfuscated. Find the right class by looking
for `ByteOrder.nativeOrder()` calls related to pixel buffers (class containing
`getPixelRGBA`/`setPixelRGBA` methods).

**Modify the bytecode:**
- Decompile with CFR or Procyon to find the exact method
- Change `invokestatic ByteOrder.nativeOrder()` to `getstatic ByteOrder/LITTLE_ENDIAN`
- Use a bytecode editor or ASM to apply the change
- Repack the jar

**Alternative: JVM agent approach** (no jar modification):
Create a small Java agent that intercepts `NativeImage` constructor calls
and forces the pixel buffer to LITTLE_ENDIAN order after construction.

### Fix 2: Test with OpenGL pixel store (User said didn't work)
```bash
# Add to JVM args:
-Dorg.lwjgl.opengl.GL_UNPACK_SWAP_BYTES=true
```
