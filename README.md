Bazel, by default, as at 6.0.0, still relies on the system python installation.

This is non-hermetic and problematic when you need to be specific about python versions; i.e. _which_ 3._x_?

Even building from source is problematic as the build system may have different library versions linked; e.g. openssl, zlib.

There are a couple of approaches others have taken to solve this:

- [Hermetic Python with Bazel](https://thethoughtfulkoala.com/posts/2020/05/16/bazel-hermetic-python.html)
- [Hermetic Python Toolchain with Bazel](https://www.anthonyvardaro.com/blog/hermetic-python-toolchain-with-bazel)
- [Dropbox](https://github.com/dropbox/dbx_build_tools/blob/master/thirdparty/cpython/BUILD.python39)

Using some form of [portable python](https://pypi.org/project/portable-python/) is maddeningly recursive as it's written in python.

Though it seems since the publication of the above, `@rules_python` has solved the problem using https://github.com/indygreg/python-build-standalone

To run this example:

```shell
bazel run --config py36 hello   # > Hello, 3.6.15!
bazel run --config py39 hello   # > Hello, 3.9.12!
bazel run --config py311 hello  # > Hello, 3.11.1!
bazel run hello                 # > Hello, 3.10.6!
```

Happily the python standalone releases can be added to a [--distdir](https://bazel.build/reference/command-line-reference#flag--distdir)

There's only a handful of python versions hardcoded into `@rules_python//python:versions.bzl`. So one would have to DIY 
the configuration for apparently esoteric versions.

One might want to DIY it anyway to make use of your own [platforms and constraints](https://bazel.build/reference/be/platform) 
to automatically select the toolchains.

Python 3.6.15 has been DIY'd using [rules_foreign_cc](https://github.com/bazelbuild/rules_foreign_cc) warts 'n' all.