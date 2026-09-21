# Android ARM build guide (`arm64-v8a`, `armeabi-v7a`)

This is the runbook for a green native QUIC build on Android ARM. The pipeline is
already in `pom.xml`; it is not run in Netty 4.2 CI, and three pom bugs will
produce an unloadable library even if CMake and Cargo succeed.

**Green** here means:

- `libnetty_quiche42.so` exists for `arm64-v8a` and `armeabi-v7a`
- ELF machine type matches the ABI
- `DT_NEEDED` is only bionic (`libc`, `libm`, `libdl`, `liblog`) plus `libc++_shared`
- Java `System.loadLibrary("netty_quiche42")` can find that soname

Host **must** be Linux x86_64. The Android profile hardcodes

`toolchains/llvm/prebuilt/linux-x86_64`

Mac and Linux aarch64 hosts will fail at the hawtjni/configure step.

Pinned sources (do not change unless you intend to re-validate BoringSSL / quiche):

| component | property | value |
|---|---|---|
| BoringSSL | `boringsslCommitSha` | `d03dbc3e5d7de44183ff17018af22323af650fbc` |
| quiche | `quicheCommitSha` | `55886df3be579579207104c8e645825b6347a209` |
| min API | `androidMinSdkVersion` | `21` |

`androidNdkVersion` in the pom is **API level 21**, not NDK r21. Install NDK r25c.

---

## 0. Patch the pom before the first build

Do these three edits in `codec-native-quic/pom.xml` inside the `android` profile.
Skipping them is why a “successful” Maven run still fails on device.

### 0.1 JNI name must be `netty_quiche42`

`Quiche.loadNativeLibrary()` and `netty_quic_quiche.c` (`LIBRARYNAME`) load
`netty_quiche42` on Android. The profile currently sets `jniLibName` to
`netty_quiche`, so the AAR ships `libnetty_quiche.so`.

```xml
<jniLibName>netty_quiche42</jniLibName>
```

### 0.2 Link `libc++_shared` and set the CMake STL

BoringSSL is C++. Incubator later added `-lc++_shared`; this tree did not.

Replace the Android `extraLdflags` line with:

```xml
<extraLdflags>-Wl,-soname=lib${jniLibName}.so -Wl,--build-id=sha1 -Wl,--strip-debug -Wl,--exclude-libs,ALL -lm -lc++_shared</extraLdflags>
```

And add `-DANDROID_STL=c++_shared` plus `-DANDROID_PLATFORM=android-21` to the
Android cmake invocation (the `build-boringssl` antrun, `platform == android`
branch):

```xml
<arg value="-DANDROID_ABI=${androidAbi}" />
<arg value="-DANDROID_PLATFORM=android-${androidMinSdkVersion}" />
<arg value="-DANDROID_STL=c++_shared" />
<arg value="-DCMAKE_TOOLCHAIN_FILE=${env.ANDROID_NDK_HOME}/build/cmake/android.toolchain.cmake" />
<arg value="-DANDROID_NATIVE_API_LEVEL=${androidMinSdkVersion}" />
```

Ship `libc++_shared.so` from the NDK next to the JNI library (step 5).

### 0.3 NDK path: Maven property **and** environment

`${ANDROID_NDK_HOME}` is a Maven property. `export ANDROID_NDK_HOME=...` is
**not** enough. Change `ndkToolchain` and the cmake toolchain file to the env
form, and still pass `-D` as a belt-and-suspenders:

```xml
<ndkToolchain>${env.ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64</ndkToolchain>
```

Keep `iterator-maven-plugin` at **0.5.0**. 0.5.1 merges nested properties and
breaks `${cargoTarget}` / `${quicheTarget}`.

### 0.4 cargo-ndk command (only if install is 4.x)

The pom currently runs:

```text
cargo ndk --target=${quicheTarget} --platform 21 -- build -p quiche --features "..." --release
```

That `--` before `build` is cargo-ndk 2 style. If `cargo ndk -V` reports 4.x and
the build dies at the cargo step, change it to:

```text
cargo ndk --target ${quicheTarget} --platform ${androidMinSdkVersion} build -p quiche --features "ffi qlog custom-client-dcid" --release
```

Do **not** use `-p 21` for the API level. In cargo-ndk 4, `-p` is cargo
`--package`. Pin 3.5.4 in step 1 so you can leave the pom command alone.

---

## 1. Toolchain

Run from a Linux x86_64 shell. Use JDK 11 for the Maven/AAR step (Netty CI
image; `android-maven-plugin` 4.6.0 is not validated on JDK 25).

```bash
# API 21, NDK r25c — the last combo that built this stack in incubator CI
NDK_VER=25.2.9519653
SDK_VER=android-21

# Android SDK + NDK (command-line tools)
# https://developer.android.com/studio#command-line-tools-only
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
mkdir -p "$ANDROID_HOME"
yes | sdkmanager --sdk_root="$ANDROID_HOME" \
  "platforms;$SDK_VER" "ndk;$NDK_VER" "platform-tools"

export ANDROID_NDK_HOME="$ANDROID_HOME/ndk/$NDK_VER"
export ANDROID_NDK="$ANDROID_NDK_HOME"
export PATH="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH"

# Host build tools
sudo apt-get install -y ninja-build cmake git clang pkg-config autoconf automake libtool
# cmake >= 3.22 is required by current BoringSSL

# Rust
curl https://sh.rustup.rs -sSf | sh -s -- -y
source "$HOME/.cargo/env"
rustup target add aarch64-linux-android armv7-linux-androideabi

# Pin cargo-ndk. Unpinned `cargo install` tracks 4.x, which broke incubator CI.
cargo install cargo-ndk --locked --version 3.5.4
```

Sanity check before Maven:

```bash
uname -m                                          # must be x86_64
cmake --version                                   # >= 3.22
ninja --version
rustc --version
cargo ndk -V                                      # 3.5.4
test -x "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang"
test -x "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi21-clang"
test -f "$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake"
```

ABI triples (clang prefix ≠ Rust target on 32-bit ARM):

| ABI | Rust `--target` | clang wrapper |
|---|---|---|
| `arm64-v8a` | `aarch64-linux-android` | `aarch64-linux-android21-clang` |
| `armeabi-v7a` | `armv7-linux-androideabi` | `armv7a-linux-androideabi21-clang` |

Use `llvm-ar` / `llvm-ranlib` / `llvm-strip`, not `armv7a-linux-androideabi-ar`.
The NDK never shipped that name ([NDK #1324](https://github.com/android/ndk/issues/1324));
the pom already does this.

---

## 2. Prove `arm64-v8a` first

From the repository root. `-DskipIteration=true` stops the iterator from also
building x86 / x86_64. `-DANDROID_NDK_HOME` is still required even after
step 0.3, because some interpolations happen before Ant reads the environment.

```bash
cd /path/to/netty

./mvnw -pl codec-native-quic -am package -DskipTests \
  -Dandroid \
  -DskipIteration=true \
  -Pandroid-arm64-v8a,android \
  -DANDROID_NDK_HOME="$ANDROID_NDK_HOME"
```

Without `-Pandroid-arm64-v8a`, `-Dandroid` on Linux auto-activates
`android-armeabi-v7a` (the default first ABI). That is step 3, not this step.

What this run does:

1. Clone BoringSSL, cmake with `ANDROID_ABI=arm64-v8a`, `ninja crypto ssl`
2. Clone quiche, rewrite `boring = { version = "4.3" }` → `"5"`, then
   `cargo ndk --target=aarch64-linux-android --platform 21 -- build -p quiche ...`
   with `BORING_BSSL_PATH` / `BORING_BSSL_INCLUDE_PATH` pointing at the prebuilt
   static libs (so `boring-sys` does not CMake BoringSSL a second time)
3. hawtjni links `libnetty_quiche42.so` with the NDK clang

Expected artifacts:

```text
codec-native-quic/target/boringssl/arm64-v8a/build/libssl.a
codec-native-quic/target/boringssl/arm64-v8a/build/libcrypto.a
codec-native-quic/target/quiche/arm64-v8a/build/libquiche.a
codec-native-quic/target/native-lib-only/arm64-v8a/META-INF/native/.../libnetty_quiche42.so
codec-native-quic/target/android-build/native-libs/arm64-v8a/libnetty_quiche42.so
```

If maven reports “BoringSSL was already build, skipping”, the skip is keyed on
the existence of `target/boringssl/<abi>`. Delete that directory (and
`target/quiche/<abi>`) after a failed half-build:

```bash
rm -rf codec-native-quic/target/boringssl/arm64-v8a \
       codec-native-quic/target/quiche/arm64-v8a \
       codec-native-quic/target/boringssl-source \
       codec-native-quic/target/quiche-source
```

---

## 3. Then `armeabi-v7a`

Same tree, do **not** `mvn clean` if you still want the arm64 artifacts for the
AAR. The skip logic is per-ABI directory.

```bash
./mvnw -pl codec-native-quic -am package -DskipTests \
  -Dandroid \
  -DskipIteration=true \
  -DANDROID_NDK_HOME="$ANDROID_NDK_HOME"
```

No extra `-P`: `android-armeabi-v7a` activates from `-Dandroid` on Linux.

Expected:

```text
codec-native-quic/target/android-build/native-libs/armeabi-v7a/libnetty_quiche42.so
```

Both ABI directories should now be populated. A subsequent package without
`skipIteration` is optional and will also compile x86 / x86_64; that is not
required for a green ARM build.

---

## 4. Copy `libc++_shared.so` into the AAR layout

The NDK shared STL is not produced by our link line; it must be packaged or
the app will fail at load with `libc++_shared.so not found`.

```bash
NDK_SYSROOT_LIB="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib"

cp -v "$NDK_SYSROOT_LIB/aarch64-linux-android/libc++_shared.so" \
  codec-native-quic/target/android-build/native-libs/arm64-v8a/

cp -v "$NDK_SYSROOT_LIB/arm-linux-androideabi/libc++_shared.so" \
  codec-native-quic/target/android-build/native-libs/armeabi-v7a/
```

If `android-maven-plugin` already wrote the AAR, re-run `package` (no `clean`)
so those files are picked up from `nativeLibrariesDirectory`.

---

## 5. Verify (this is the green check)

`READELF` is the NDK llvm one, not the host GNU one.

```bash
READELF="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-readelf"

so64=codec-native-quic/target/android-build/native-libs/arm64-v8a/libnetty_quiche42.so
so32=codec-native-quic/target/android-build/native-libs/armeabi-v7a/libnetty_quiche42.so

test -f "$so64" && test -f "$so32"

"$READELF" -h "$so64" | grep -E 'Class:|Machine:'
# Class: ELF64
# Machine: AArch64

"$READELF" -h "$so32" | grep -E 'Class:|Machine:'
# Class: ELF32
# Machine: ARM

"$READELF" -d "$so64" | grep NEEDED
"$READELF" -d "$so32" | grep NEEDED
# accept: libc.so, libm.so, libdl.so, liblog.so, libc++_shared.so
# reject: libstdc++.so.6, libgcc_s.so.1, ld-linux-*.so, libssl.so, libcrypto.so
```

Static `libssl.a` / `libcrypto.a` / `libquiche.a` must not appear as `DT_NEEDED`.
If they do, the JNI link used shared objects instead of the `-L... -lssl`
archives.

Optional on-device load (API ≥ 21, matching ABI):

```bash
adb push "$so64" /data/local/tmp/libnetty_quiche42.so
adb shell 'cd /data/local/tmp && LD_LIBRARY_PATH=. toybox readelf -d libnetty_quiche42.so'
```

A full `Quiche.ensureAvailability()` check needs a Dalvik/ART process and the
classes JAR; ELF + soname is the native gate.

---

## 6. Optional: both ARM ABIs in one Maven invocation

The iterator always adds x86 and x86_64 after the auto-activated v7a first
pass. For ARM-only, keep using steps 2 and 3.

To build all four ABIs (slow, re-clones BoringSSL/quiche per ABI because the
antrun deletes the source trees after each success):

```bash
./mvnw -pl codec-native-quic -am package -DskipTests \
  -Dandroid \
  -DANDROID_NDK_HOME="$ANDROID_NDK_HOME"
```

Do not bump `iterator-maven-plugin` past 0.5.0.

---

## 7. Standalone fallback (no Maven)

Use this only to isolate a CMake or cargo-ndk failure. The JNI `.so` still
needs hawtjni or an equivalent clang link of `src/main/c/*.c` against the
static libs.

```bash
# --- BoringSSL arm64-v8a ---
git clone --branch main --single-branch \
  https://boringssl.googlesource.com/boringssl boringssl-source
git -C boringssl-source checkout d03dbc3e5d7de44183ff17018af22323af650fbc

cmake -S boringssl-source -B boringssl-source/build/arm64-v8a -GNinja \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-21 \
  -DANDROID_STL=c++_shared \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake" \
  -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE \
  -DCMAKE_BUILD_TYPE=Release
ninja -C boringssl-source/build/arm64-v8a crypto ssl

# --- quiche arm64-v8a ---
git clone --branch master --single-branch \
  https://github.com/cloudflare/quiche quiche-source
git -C quiche-source checkout 55886df3be579579207104c8e645825b6347a209
sed -i 's/boring = { version = "4.3" }/boring = { version = "5" }/' \
  quiche-source/Cargo.toml

mkdir -p staging/arm64-v8a/build staging/arm64-v8a/include
cp boringssl-source/build/arm64-v8a/libssl.a \
   boringssl-source/build/arm64-v8a/libcrypto.a staging/arm64-v8a/build/
cp -a boringssl-source/include/. staging/arm64-v8a/include/

( cd quiche-source && \
  BORING_BSSL_PATH="$PWD/../staging/arm64-v8a/build" \
  BORING_BSSL_INCLUDE_PATH="$PWD/../staging/arm64-v8a/include" \
  cargo ndk -t arm64-v8a --platform 21 build -p quiche \
    --features "ffi qlog custom-client-dcid" --release )

# Repeat with ANDROID_ABI=armeabi-v7a, cargo ndk -t armeabi-v7a,
# and copy from boringssl-source/build/armeabi-v7a/.
```

If cmake puts `libssl.a` under `ssl/` and `libcrypto.a` under `crypto/` instead
of the build root, copy from those subdirs. `boring-sys` searches `lib/`,
`crypto/`, `ssl/`, and the `BORING_BSSL_PATH` directory itself.

---

## 8. Failures seen before, and what to do

| Symptom | Cause | Action |
|---|---|---|
| cmake: toolchain file not found, path starts with `/build/cmake/...` | `${ANDROID_NDK_HOME}` empty as a Maven property | pass `-DANDROID_NDK_HOME=$ANDROID_NDK_HOME`; apply step 0.3 |
| `armv7a-linux-androideabi-ar: No such file` | using the clang triple for binutils | use `llvm-ar` (already in the pom) |
| cargo-ndk segfault / “unknown package: 21” | cargo-ndk 4.x CLI | pin 3.5.4, or drop `--` before `build` and never use `-p 21` |
| `UnsatisfiedLinkError: netty_quiche42` | pom still names the `.so` `netty_quiche` | step 0.1 |
| `libc++_shared.so not found` | STL not linked / not packaged | steps 0.2 and 4 |
| BoringSSL / quiche “already build, skipping” after a failed ABI | leftover `target/boringssl/<abi>` or `target/quiche/<abi>` | delete those dirs, not necessarily all of `target/` |
| hawtjni cannot find `libnetty_quiche42.so` without a version suffix | Android libtool `library_names_spec` | already handled by `src/main/native-package/m4/custom.m4.android.template` |
| build on Mac / aarch64 Linux | `ndkToolchain` is `linux-x86_64` | use a Linux x86_64 host or docker `--platform linux/amd64` |
| iterator builds the wrong ABI | plugin 0.5.1 | stay on 0.5.0 |

Incubator skipped Android CI because cargo-ndk segfaulted
([netty-incubator-codec-quic#822](https://github.com/netty/netty-incubator-codec-quic/pull/822)).
Treat an unpinned `cargo install cargo-ndk` as the first suspect.

---

## 9. Order of work (checklist)

1. Apply section 0 pom edits; keep iterator-maven-plugin 0.5.0.
2. Install NDK `25.2.9519653`, Ninja, CMake ≥ 3.22, rustup targets, cargo-ndk **3.5.4**.
3. `arm64-v8a` only (`-DskipIteration=true -Pandroid-arm64-v8a,android`).
4. Confirm `libnetty_quiche42.so` is ELF64 / AArch64 (section 5).
5. `armeabi-v7a` only (`-DskipIteration=true -Dandroid`).
6. Confirm `libnetty_quiche42.so` is ELF32 / ARM.
7. Copy `libc++_shared.so` into both `native-libs/<abi>/` dirs.
8. Re-package if you need the AAR; skip the four-ABI iterator until ARM is green.
