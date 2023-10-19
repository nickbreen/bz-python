Bazel, as at 6.0.0, still relies on, by default, the system python installation.

This is non-hermetic and problematic when you need to be specific about python versions; i.e. _which_ 3._x_?

# Python Rules

Happily the python standalone releases can be added to a [--distdir](https://bazel.build/reference/command-line-reference#flag--distdir)

There's only a handful of python versions hardcoded into `@rules_python//python:versions.bzl`. So one would have to DIY 
the configuration for apparently esoteric versions.

# Building

One might want to DIY it anyway and make use of your own [platforms and constraints](https://bazel.build/reference/be/platform)
to automatically select the toolchain(s).

Building from source is problematic as the build system may have different library versions linked; e.g. openssl, zlib.

There are a couple of approaches others have taken to solve this:

- [Hermetic Python with Bazel](https://thethoughtfulkoala.com/posts/2020/05/16/bazel-hermetic-python.html)
- [Hermetic Python Toolchain with Bazel](https://www.anthonyvardaro.com/blog/hermetic-python-toolchain-with-bazel)
- [Dropbox](https://github.com/dropbox/dbx_build_tools/blob/master/thirdparty/cpython/BUILD.python39)

Using some form of [portable python](https://pypi.org/project/portable-python/) is maddeningly recursive as it's 
written _in python_.

## Overview

Python 3.6.15 has been DIY'd with [rules_foreign_cc](https://github.com/bazelbuild/rules_foreign_cc) warts 'n' all.

- You'll need the python source for which ever version you want, this example uses 3.6.x as that happens to be the version 
  available in crappy old EoL'd RHEL6 which just might _still_ be hanging around smelling up the place.
- You'll also need sources for some python modules' dependencies: this example uses zlib, bzip2, and openssl as you'll need
  these to use `pip`.
- Load these in `WORKSPACE` with `http_archive`, build them using `rules_foreign_cc`.
- Define a `py_runtime_pair` toolchain and use it.

## Inspecting Python Modules

```shell
bz2_dirs=($(bazel cquery --output files @bzip2-1.0.8//:dir))
ssl_dirs=($(bazel cquery --output files @openssl-1.1.1w//:dir))
py3_dirs=($(bazel cquery --output files @python-3.6.15//:dir))

# What do they need?
objdump -p ${py3_dirs[0]}/lib/python3.6/lib-dynload/_bz2.cpython-36m-x86_64-linux-gnu.so | grep NEEDED
objdump -p ${py3_dirs[0]}/lib/python3.6/lib-dynload/_ssl.cpython-36m-x86_64-linux-gnu.so | grep NEEDED
# Where can they be found?
export LD_LIBRARY_PATH="${bz2_dirs[@]/%//lib:}${ssl_dirs[@]/%//lib:}${LD_LIBRARY_PATH}"
ldd ${py3_dirs[0]}/lib/python3.6/lib-dynload/_bz2.cpython-36m-x86_64-linux-gnu.so
ldd ${py3_dirs[0]}/lib/python3.6/lib-dynload/_ssl.cpython-36m-x86_64-linux-gnu.so
```

## Problems

Notice (above) we have to configure `LD_LIBRARY_PATH` manually, the same applies to `genrule` and `py_binary` rules.

Dynamic linking is fiddly. Configuring python to static link is also fiddly, if this works at all:

```BUILD
genrule(
    name = "local",  # ... and include in :srcs
    outs = ["python36/Modules/Setup.local"],  # path prefix includes the value of configure_make(lib_name) 
    cmd = """
    exec >> $@
    # Copied from Modules/Setup.dist and modified
    echo 'SSL=$(location @openssl//:dir)'
    echo '_ssl _ssl.c -DUSE_SSL -I$$(SSL)/include -I$$(SSL)/include/openssl $$(SSL)/lib/libssl.a $$(SSL)/lib/libcrypto.a  # statically link in libssl and libcrypto'
    """,
    tools = [
        "@openssl//:dir",
    ],
)
```

# Examples

To run these examples:

```shell
bazel test --config py36 ...
bazel test --config py39 ...
bazel test --config py311 ...
bazel test ...
```

```shell
bazel run --config py36 hello   # > Hello, 3.6.15!       # this is our DIY built toolchain.
bazel run --config py39 hello   # > Hello, 3.9.12!
bazel run --config py311 hello  # > Hello, 3.11.1!
bazel run hello                 # > Hello, 3.10.7!       # this is the OS's python version.
```

```shell
bazel run --config py36 hello_ssl   # > Hello SSL, OpenSSL 1.1.1t  7 Feb 2023!       # this is our DIY built toolchain.
bazel run --config py39 hello_ssl   # > Hello SSL, OpenSSL 1.1.1n  15 Mar 2022!
bazel run --config py311 hello_ssl  # > Hello SSL, OpenSSL 1.1.1s  1 Nov 2022!
bazel run hello_ssl                 # > Hello SSL, OpenSSL 3.0.5 5 Jul 2022!         # this is the OS's openssl version.
```

# Problems

The zipimport module requires zlib, specifically the zlib sources (zlib1g-dev or zliv-devel).
