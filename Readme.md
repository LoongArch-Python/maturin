# Pyo3-pack

[![Linux and Mac Build Status](https://img.shields.io/travis/PyO3/pyo3-pack/master.svg?style=flat-square)](https://travis-ci.org/PyO3/pyo3-pack)
[![Windows Build status](https://img.shields.io/appveyor/ci/konstin/pyo3-pack/master.svg?style=flat-square)](https://ci.appveyor.com/project/konstin/pyo3-pack/branch/master)
[![Crates.io](https://img.shields.io/crates/v/pyo3-pack.svg?style=flat-square)](https://crates.io/crates/pyo3-pack)
[![PyPI](https://img.shields.io/pypi/v/pyo3-pack.svg?style=flat-square)](https://pypi.org/project/pyo3-pack/)
[![Chat on Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?style=flat-square)](https://gitter.im/PyO3/Lobby)

Build and publish crates with pyo3, rust-cpython and cffi bindings as well as rust binaries as python packages.

This project was meant as a zero configuration replacement for [setuptools-rust](https://github.com/PyO3/setuptools-rust). It supports building wheels for python 3.5+ on windows, linux and mac, can upload them to [pypi](https://pypi.org/) and has basic pypy support.

## Usage

You can either download binaries from the [latest release](https://github.com/PyO3/pyo3-pack/releases/latest) or install it with pip:

```shell
pip install pyo3-pack
```

You can also install pyo3-pack from source, though it's an older version:

```shell
cargo install pyo3-pack
```

There are three main commands:

 * `pyo3-pack publish` builds the crate into python packages and publishes them to pypi.
 * `pyo3-pack build` builds the wheels and stores them in a folder (`target/wheels` by default), but doesn't upload them.
 * `pyo3-pack develop` builds the crate and install it's as a python module directly in the current virtualenv.

`pyo3` and `rust-cpython` bindings are automatically detected, for cffi or binaries you need to pass `-b cffi` or `-b bin`. pyo3-pack needs no extra configuration files, and also doesn't clash with an existing setuptools-rust or milksnake configuration. You can even integrate it with testing tools such as tox (see `pyo3-pure` for an example).

The name of the package will be the name of the cargo project, i.e. the name field in the `[package]` section of Cargo.toml. The name of the module, which you are using when importing, will be the `name` value in the `[lib]` section (which defaults to the name of the package). For binaries it's simply the name of the binary generated by cargo.

Pip allows adding so called console scripts, which are shell commands that execute some function in you program. You can add console scripts in a section `[package.metadata.pyo3-pack.scripts]`. The keys are the script names while the values are the path to the function in the format `some.module.path:class.function`, where the `class` part is optional. The function is called with no arguments. Example:

```toml
[package.metadata.pyo3-pack.scripts]
get_42 = "my_project:DummyClass.get_42"
```

You can also specify [trove classifiers](https://pypi.org/classifiers/) in your Cargo.toml under `package.metadata.pyo3-pack.classifier`, e.g.:

```toml
[package.metadata.pyo3-pack]
classifier = ["Programming Language :: Python"]
```

## pyo3 and rust-cpython

For pyo3 and rust-cpython, pyo3-pack can only build packages for installed python versions. On linux and mac, all python versions in `PATH` are used. If you don't set your own interpreters with `-i`, a heuristic is used to search for python installations. On windows all versions from the python launcher (which is installed by default by the python.org installer) and all conda environments except base are used. You can check which versions are picked up with the `list-python` subcommand.

pyo3 will set the used python interpreter in the environment variable `PYTHON_SYS_EXECUTABLE`, which can be used from custom build scripts.

## Cffi

 Cffi wheels are compatible with all python versions, but they need to have `cffi` installed for the python used for building (`pip install cffi`).

pyo3-pack will run cbindgen and generate cffi bindings. You can override this with a build script that writes a header to `target/header.h`.

<details>
<summary>Example of a custom build script</summary>

```rust
use cbindgen; // Use `extern crate cbindgen` in rust 2015
use std::env;
use std::path::Path;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let bindings = cbindgen::Builder::new()
        .with_no_includes()
        .with_language(cbindgen::Language::C)
        .with_crate(crate_dir)
        .generate()
        .unwrap();
    bindings.write_to_file(Path::new("target").join("header.h"));
}
```

</details>

## Mixed rust/python projects

To create a mixed rust/python project, create a folder with your module name (i.e. `lib.name` in Cargo.toml) next to your Cargo.toml and add your python sources there:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   └── bar.py
├── Readme.md
└── src
    └── lib.rs
```

pyo3-pack will add the native extension as a module in your python folder. When using develop, pyo3-pack will copy the native library and for cffi also the glue code to your python folder. You should add those files to your gitignore.

With cffi you can do `from .my_project import lib` and then use `lib.my_native_function`, with pyo3/rust-cpython you can directly `from .my_project import my_native_function`.

Example layout with pyo3 after `pyo3-pack develop`: 

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   ├── bar.py
│   └── my_project.cpython-36m-x86_64-linux-gnu.so
├── Readme.md
└── src
    └── lib.rs
```

## Manylinux and auditwheel

For portability reasons, native python modules on linux must only dynamically link a set of very few libraries which are installed basically everywhere, hence the name manylinux. The pypa offers a special docker container and a tool called [auditwheel](https://github.com/pypa/auditwheel/) to ensure compliance with the [manylinux rules](https://www.python.org/dev/peps/pep-0513/#the-manylinux1-policy).

pyo3-pack contains a reimplementation of a major part of auditwheel automatically checking the generated library. If you want to disable those checks or build for native linux target, use the `--manylinux` flag.

For full manylinux compliance you need to compile in a cent os 5 docker container. The [konstin2/pyo3-pack](https://hub.docker.com/r/konstin2/pyo3-pack) image is based on the official manylinux image. You can use it like this:

```
docker run --rm -v $(pwd):/io konstin2/pyo3-pack build
```

pyo3-pack itself is manylinux compliant when compiled for the musl target. The binaries on the release pages have additional keyring integration (through the `password-storage` feature), which is not manylinux compliant.

## PyPy

pyo3-pack can build wheels for pypy with pyo3. Note that pypy support in pyo3 is unreleased as of this writing. Also note that pypy [is not compatible with manylinux1](https://github.com/antocuni/pypy-wheels#why-not-manylinux1-wheels) and you can't publish pypy wheel to pypi pypy has been only tested manually and on linux. See [#115](https://github.com/PyO3/pyo3-pack/issues/115) for more details.

### Build

```
FLAGS:
    -h, --help
            Prints help information

        --release
            Pass --release to cargo

        --skip-auditwheel
            [deprecated, use --manylinux instead] Don't check for manylinux compliance

        --strip
            Strip the library for minimum file size

    -V, --version
            Prints version information


OPTIONS:
    -m, --manifest-path <PATH>
            The path to the Cargo.toml [default: Cargo.toml]

        --target <TRIPLE>
            The --target option for cargo

    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
        --manylinux <manylinux>
            Control the platform tag on linux.

            - `1`: Use the manylinux1 tag and check for compliance

            - `1-unchecked`: Use the manylinux1 tag without checking for compliance

            - `2010`: Use the manylinux2010 tag and check for compliance

            - `2010-unchecked`: Use the manylinux1 tag without checking for compliance

            - `off`: Use the native linux tag (off)

            This option is ignored on all non-linux platforms [default: 1]  [possible values: 1, 1-unchecked, 2010,
            2010-unchecked, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
```

### Publish

```
FLAGS:
        --debug
            Do not pass --release to cargo

    -h, --help
            Prints help information

        --no-strip
            Strip the library for minimum file size

        --skip-auditwheel
            [deprecated, use --manylinux instead] Don't check for manylinux compliance

    -V, --version
            Prints version information


OPTIONS:
    -m, --manifest-path <PATH>
            The path to the Cargo.toml [default: Cargo.toml]

        --target <TRIPLE>
            The --target option for cargo

    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
        --manylinux <manylinux>
            Control the platform tag on linux.

            - `1`: Use the manylinux1 tag and check for compliance

            - `1-unchecked`: Use the manylinux1 tag without checking for compliance

            - `2010`: Use the manylinux2010 tag and check for compliance

            - `2010-unchecked`: Use the manylinux1 tag without checking for compliance

            - `off`: Use the native linux tag (off)

            This option is ignored on all non-linux platforms [default: 1]  [possible values: 1, 1-unchecked, 2010,
            2010-unchecked, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
    -p, --password <password>
            Password for pypi or your custom registry. Note that you can also pass the password through
            PYO3_PACK_PASSWORD
    -r, --repository-url <registry>
            The url of registry where the wheels are uploaded to [default: https://upload.pypi.org/legacy/]

        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
    -u, --username <username>
            Username for pypi or your custom registry
```

### Develop

```
FLAGS:
    -h, --help
            Prints help information

        --release
            Pass --release to cargo

        --strip
            Strip the library for minimum file size

    -V, --version
            Prints version information


OPTIONS:
    -b, --bindings <binding_crate>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo_extra_args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -m, --manifest-path <manifest_path>
            The path to the Cargo.toml [default: Cargo.toml]

        --rustc-extra-args <rustc_extra_args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
```

## Code

The main part is the pyo3-pack library, which is completely documented and should be well integratable. The accompanying `main.rs` takes care username and password for the pypi upload and otherwise calls into the library.

The `sysconfig` folder contains the output of `python -m sysconfig` for different python versions and platform, which is helpful during development.

You need to install `cffi` (`pip install cffi`) to run the tests.

You might want to have look into my [blog post](https://blog.schuetze.link/2018/07/21/a-dive-into-packaging-native-python-extensions.html) which explains the intricacies of building native python packages.
