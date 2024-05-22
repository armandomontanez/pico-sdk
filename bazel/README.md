# Bazel build
The Bazel build for the Pico SDK is currently community-maintained, and should
be considered an experimental work-in-progress. There are missing features,
and you may encounter significant breakages with future versions.

You are welcome and encouraged to file issues for any problems you encounter
along the way.

## Using the Pico SDK in a Bazel project.

### Add pico-sdk as a dependency
First, in your `MODULE.bazel` file, add a dependency on the Pico SDK:
```python
bazel_dep(
  name = "pico-sdk",
  version = "1.6.0-rc1",
)
```
Second, in the same file you'll need to add an explicit dependency on
`rules_cc`, as it's a special-cased Bazel module:
```python
# Note: rules_cc is special-cased repository; a dependency on rules_cc in a
# module will not ensure that the root Bazel module has that same version of
# rules_cc. For that reason, this primarily acts as a FYI. You'll still need
# to explicitly list this dependency in your own project's MODULE.bazel file.
bazel_dep(name = "rules_cc", version = "0.0.10")

# rules_cc v0.0.10 is not yet cut, so manually pull in the desired version.
# This does not apply to dependent projects, so it needs to be copied to your
# project's MODULE.bazel too.
archive_override(
    module_name = "rules_cc",
    urls = "https://github.com/bazelbuild/rules_cc/archive/1acf5213b6170f1f0133e273cb85ede0e732048f.zip",
    strip_prefix = "rules_cc-1acf5213b6170f1f0133e273cb85ede0e732048f",
    integrity = "sha256-NddP6xi6LzsIHT8bMSVJ2NtoURbN+l3xpjvmIgB6aSg=",
)
```

### Register toolchains
These toolchains tell Bazel how to compile for ARM cores. Add the following
to the `MODULE.bazel` for your project:
```python
register_toolchains(
    "@pico-sdk//bazel/toolchain:arm_gcc_linux-x86_64",
    "@pico-sdk//bazel/toolchain:arm_gcc_win-x86_64",
    "@pico-sdk//bazel/toolchain:arm_gcc_mac-x86_64",
    "@pico-sdk//bazel/toolchain:arm_gcc_mac-aarch64",
)
```

### Enable required .bazelrc flags
To use the toolchains provided by the Pico SDK, you'll need to enable a few
new features. In your project's `.bazelrc`, add the following
```
# Required for new toolchain resolution API.
build --incompatible_enable_cc_toolchain_resolution
build --@rules_cc//cc/toolchains:experimental_enable_rule_based_toolchains
```

### Ready to build!
You're now ready to start building Pico Projects in Bazel! When building,
don't forget to specify `--platforms` so Bazel knows you're targeting the
Raspberry Pi Pico:
```console
$ bazelisk build --platforms=@pico-sdk//bazel/platform:rp2040 //...
```

## SDK configuration [experimental]
These configuration options are a work in progress and may see significant
breaking changes in future versions.

### Selecting a different board
Currently there are three configurable flags for targeting a different board:
1. `pico_config_extra_headers`: This should always point to a `cc_library `that
   provides a `"pico_config_extra_headers.h"` header. You can configure this
   by including a flag like the following in your build invocation:
   ```
   --@pico-sdk//bazel/config:pico_config_extra_headers=//path/to:custom_extra_headers
   ```
2. `pico_config_platform_headers`: This should always point to a `cc_library`
   that provides a `"pico_config_platform_headers.h"` header.
   ```
   --@pico-sdk//bazel/config:pico_config_platform_headers=//path/to:custom_platform_headers
   ```
3. `pico_config_header`: This should point to a `cc_library` that sets all
   necessary SDK defines. Most notably, `PICO_BOARD`, `PICO_CONFIG_HEADER`,
   `PICO_ON_DEVICE`, `PICO_NO_HARDWARE`, and `PICO_BUILD`. See
   `//src/boards:BUILD.bazel` for working examples. Any `defines` set on this
   library will propagate to the rest of the Pico SDK. To set this configuration
   option, pass a flag like the following in your Bazel build invocation:
   ```
   --@pico-sdk//bazel/config:pico_config_platform_headers=//path/to:pico_board_config
   ```

### Selecting a stdio mode
To select a different stdio mode, add it to your `platform` definition. For
example:
```python
platform(
    name = "rp2040",
    constraint_values = [
        "@pico-sdk//bazel/constraint:rp2040",
        "@pico-sdk//bazel/constraint:stdio_usb", # Configures stdio_mode.
        "@platforms//cpu:armv6-m",
    ],
)
```

## Building the Pico SDK itself

### First time setup
You'll need Bazel (v7.0.0 or higher) or Bazelisk (a self-updating Bazel
launcher) to build the Pico SDK.

We strongly recommend you set up
[Bazelisk](https://bazel.build/install/bazelisk).

### Building
To build all of the Pico SDK, run the following command:
```console
$ bazelisk build --platforms=//bazel/platform:rp2040 //...
```

**Note:** Since the Bazel build does not yet have any `cc_binary` rules with a
`main()` function, there won't be any binaries to flash on your board. For now,
this only builds the SDK as a collection of libraries.

## Known issues and limitations
The Bazel build is currently experimental and incomplete. At this time, only the
stock Pi Pico board is supported, and the only configuration options are
changing the STDIO mode between UART and USB serial.

Keep in mind the following limitations:
* Pico-W is not yet supported.
* Selecting an alternative board is not yet supported.
* Nearly all preexisting CMake configuration options are not yet supported.
* Targeting the host build of the Pico SDK is not yet supported.
