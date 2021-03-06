# Flutter Rust FFI Template

This project is a Flutter Plugin template. 

It provides out-of-the box support for cross-compiling native Rust code for all available iOS and Android architectures and call it from plain Dart using [Foreign Function Interface](https://en.wikipedia.org/wiki/Foreign_function_interface).

This template provides first class FFI support, **the clean way**. 
- No Swift or Kotlin wrappers
- No message channels
- No async calls
- No need to export `aar` bundles or `.framework`'s

## Getting started

### Write your native code

Edit your code within `rust/src/lib.rs` and add any dependencies you need.

Make sure to annotate your exported functions with `#[no_mangle]` and `pub extern` so the function names can be dealt with by Dart.  

Returning strings or structs may require using `unsafe` code. 

### Compile the library

- Ensure the NDK is installed
- Adapt the paths from `cargo-config.toml` to your NDK installation and append them to your `~/.cargo/config` file
- Set the name of your library in `Cargo.toml`
- On the `rust` folder, run `make init` and `make all`

Generated artifacts:
- Android libraries
  - `target/aarch64-linux-android/release/libexample.so`
  - `target/armv7-linux-androideabi/release/libexample.so`
  - `target/i686-linux-android/release/libexample.so`
- iOS library
  - `target/universal/release/libexample.a`
- Bindings header
  - `target/bindings.h`

### Reference the shared objects

#### iOS

Ensure that `rust/ios/mylib.podspec` includes the following directives:

```diff
...
   s.source           = { :path => '.' }
+  s.public_header_files = 'Classes**/*.h'
   s.source_files = 'Classes/**/*'
+  s.static_framework = true
+  s.vendored_libraries = "**/*.a"
   s.dependency 'Flutter'
   s.platform = :ios, '8.0'
...
```

On `flutter/ios`, place a symbolic link to the `libexample.a` file

```sh
$ cd flutter/ios
$ ln -s ../rust/target/universal/release/libexample.a .
```

Append the function signatures from `rust/target/bindings.h` into `flutter/ios/Classes/MylibPlugin.h`

```sh 
$ cd flutter/ios
$ cat ../rust/target/bindings.h >> Classes/MylibPlugin.h
```

In our case, it will append `char *rust_greeting(const char *to);` and `void rust_greeting_free(char *s);`

By default, XCode will skip bundling the `libexample.a` library if it detects that it is not being used. To force its inclusion, add a dummy method in `SwiftMylibPlugin.m` that uses at least one of the native functions:

```kotlin
...
  public func dummyMethodToEnforceBundling() {
    rust_greeting("");
  }
}
```

If you are not using Flutter channels, the rest of methods can be left empty.

#### Android

Similarly as we did on iOS with `libexample.a`, create symlinks pointing to the binary libraries on `rust/target`.

Create the following structure on `flutter/android` for each architecture:

```
src
`-- main
    `-- jniLibs
        |-- arm64-v8a
        |   `-- libexample.so@ -> ../../../../../rust/target/aarch64-linux-android/release/libexample.so
        |-- armeabi-v7a
        |   `-- libexample.so@ -> ../../../../../rust/target/armv7-linux-androideabi/release/libexample.so
        `-- x86
            `-- libexample.so@ -> ../../../../../rust/target/i686-linux-android/release/libexample.so
```

As before, if you are not using Flutter channels, the methods within `android/src/main/kotlin/org/mylib/mylib/MylibPlugin.kt` can be left empty.

### Declare the bindings in Dart

In `/lib/mylib.dart`, initialize the function bindings from Dart and implement any additional logic that you need.

Load the library: 
```dart
final DynamicLibrary nativeExampleLib = Platform.isAndroid
    ? DynamicLibrary.open("libexample.so")
    : DynamicLibrary.process();
```

Find the symbols we want to use, with the appropriate Dart signatures:
```dart
final Pointer<Utf8> Function(Pointer<Utf8>) rustGreeting = nativeExampleLib
    .lookup<NativeFunction<Pointer<Utf8> Function(Pointer<Utf8>)>>("rust_greeting")
    .asFunction();

final void Function(Pointer<Utf8>) freeGreeting = nativeExampleLib
    .lookup<NativeFunction<Void Function(Pointer<Utf8>)>>("rust_greeting_free")
    .asFunction();
```

Call them:
```dart
// Prepare the parameters
final name = "John Smith";
final Pointer<Utf8> arg1 = Utf8.toUtf8(name);
print("- Calling rust_greeting with argument:  $arg1");

// Call rust_greeting
final Pointer<Utf8> resultPointer = rustGreeting(argName);
print("- Result pointer:  $resultPointer");

final String greetingStr = Utf8.fromUtf8(resultPointer);
print("- Response string:  $greetingStr");
```

When we are done using `greetingStr`, tell Rust to free it, since the Rust implementation kept it alive for us to use it.
```dart
freeGreeting(resultPointer);
```

## More information
- https://flutter.dev/docs/development/platform-integration/c-interop
- https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-06-rust-on-ios.html
- https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html
- https://github.com/dart-lang/samples/blob/master/ffi/structs/structs.dart
