---
layout: post
title: Nix & libSystem
tags: nix macos
---

One of the main selling points of the Nix package manager is it's *reproducible builds*,
this means different things depending on the context but the basic premise is the following.

> Anything that's not explicitly referenced in a *derivation* shouldn't influence the result of
> builds, regardless of what libraries are installed on the system or kernel version it's running.

This is achieved by isolating builds as much as possible, only giving access to
libraries and tools explicitly specified.  The default build environment in Nix
is completely empty, no compilers, no bash, no libc.  Everything available in the
*stdenv* is built up from nothing.

This is where some problems arise.

## opensource.apple.com

Since macOS is based on darwin many things present on a default installation are opensource. This
makes it possible for Nix to build these components from source regardless of what version is
present on the particular version of macOS the machine is running.

At the time of writing nixpkgs is still compatible with 10.12 and uses
[Apple's 10.12.6 sources][apple-source-releases] for most of the components which can be found at
[opensource.apple.com](https://opensource.apple.com).

```nix
Libm        = applePackage "Libm"        "osx-10.7.4"  "02sd82ig2jvvyyfschmb4gpz6psnizri8sh6i982v341x6y4ysl7" {};
Libnotify   = applePackage "Libnotify"   "osx-10.12.6" "0p5qhvalf6j1w6n8xwywhn6dvbpzv74q5wqrgs8rwfpf74wg6s9z" {};
libplatform = applePackage "libplatform" "osx-10.12.6" "0rh1f5ybvwz8s0nwfar8s0fh7jbgwqcy903cv2x8m15iq1x599yn" {};
libpthread  = applePackage "libpthread"  "osx-10.12.6" "1j6541rcgjpas1fc77ip5krjgw4bvz6jq7bq7h9q7axb0jv2ns6c" {};
libresolv   = applePackage "libresolv"   "osx-10.12.6" "077j6ljfh7amqpk2146rr7dsz5vasvr3als830mgv5jzl7l6vz88" {};
...
```

## libSystem

However, this isn't possible for everything, macOS is proprietary software so not
everything is openly available. This includes libSystem.

So what is libSystem exactly?

```bash
$ otool -L /usr/bin/true
/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

On linux there's a clear separation between libc and the kernel but libSystem is an amalgamation
of many libraries, one of which is libc.  This means Nix builds must also link against this system
library, introducing an impurity into the build environment. Running a build on one version of macOS
might have more symbols available in libSystem compared to another.  And what's worse cached
binaries built using one machine might fail to run on another because of this.

Libsystem itself is an *umbrella* library which just re-exports symbols from various other
libraries.  The same mechanism can be used to create a Nix variant of the *umbrella* that only
exports symbols available on all supported macOS versions.

```bash
mkdir -p $out/lib/system
ld -dylib -o $out/lib/system/libsystem_c.dylib \
   /usr/lib/libSystem.dylib \
   -reexported_symbols_list ${./system_c_symbols}
ld -dylib -o $out/lib/system/libsystem_kernel.dylib \
   /usr/lib/libSystem.dylib \
   -reexported_symbols_list ${./system_kernel_symbols}
```

This is used in the [`darwin.Libsystem`][libsystem] package to provide a shim for libSystem that
pretends to be an older version than it is in reality, while sill using `/usr/lib/libSystem.B.dylib`.

```bash
$ otool -L /nix/store/m5ajgnzp2512na31brwfmydwk3l1gawb-coreutils-8.31/bin/true
/nix/store/1v27jz329ngx1s7i3qvb9r7gfk73k2jx-gmp-6.2.0/lib/libgmp.10.dylib (compatibility version 15.0.0, current version 15.0.0)
/nix/store/nb3ag6cn5s856vykaiaz43qpc0ajkira-Libsystem-osx-10.12.6/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1226.10.1)
```

This approach does have one problem. Because of the way libSystem is built up the Nix umbrella also
links against various sub libraries of libSystem located in `/usr/lib/system`. If one of these is
removed in a macOS update our [reexported_libraries][reexported_libraries] list needs to be updated
to make Nix packages work on the new version as they will try to load non existent libraries.

## MACOSX_DEPLOYMENT_TARGET

Setting the `MACOSX_DEPLOYMENT_TARGET` environment variable or using the `-mmacosx-version-min`
linker flag is a way to indicate the desired compatibility for a project.  This is used in many of
Apple's headers as a condition to enable certain features in the SDK so developers can target old
macOS versions using the latest toolchain.  This is useful, but as far as I'm aware this doesn't
solve the problem with libSystem symbols described in the section above.

Since nixpkgs 20.03 this along with some other flags are set in the standard environment making most
binaries binary reproducible across machines. Using the following linker flags
`-macosx_version_min 10.12 -sdk_version 10.12 -no_uuid` none of these system inconsistencies end up
in the binary anymore.  Meaning that building simple packages locally will result in exactly the
same output as [cache.nixos.org](https://cache.nixos.org) or other binary caches.

```diff
 Load command 8
      cmd LC_UUID
  cmdsize 24
-    uuid E0DB2092-9D68-378E-919B-11F4FBC178C8
+    uuid 4490662C-1618-3452-91F7-B931CA6CE292
 Load command 9
       cmd LC_VERSION_MIN_MACOSX
   cmdsize 16
   version 10.10
-      sdk 10.12
+      sdk 10.13
 Load command 10
       cmd LC_SOURCE_VERSION
   cmdsize 16
```

[apple-source-releases]: https://github.com/NixOS/nixpkgs/blob/bdb59380a39d63fb3b0f22b90f3fcaf40686e0d7/pkgs/os-specific/darwin/apple-source-releases/default.nix
[libsystem]: https://github.com/NixOS/nixpkgs/blob/bdb59380a39d63fb3b0f22b90f3fcaf40686e0d7/pkgs/os-specific/darwin/apple-source-releases/Libsystem/default.nix
[reexported_libraries]: https://github.com/NixOS/nixpkgs/blob/bdb59380a39d63fb3b0f22b90f3fcaf40686e0d7/pkgs/os-specific/darwin/apple-source-releases/Libsystem/reexported_libraries
