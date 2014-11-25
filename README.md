### Informal Goals

* a build process is defined by package file(s)
* package files are declarative, thus, order does not matter
* package files end with `.bp`.
* should provide a fast way to build complex dependencies
* support high build-time concurrency within one runtime process (e.g. not forking)
* pb files are located in project root (searched from cwd towards rootdir)
* pb manages intermediate and target files in its own directory
* easy specification of optional & required dependencies
* supports different target builders
  * C, C++ must be supported in initial milestone (immediate usecases)
    * executable files (target type: `executable`)
    * shared objects (target type: `library`)
    * static libraries (target type: `archive`)
  * asset bundling (minifying, merging, uglifying) next step
    * target type: hmm... maybe assets?
    * should support cache buster and manifest generations/generators.
  * Scala/Java
    * target type: `jar`
* the run processor actually watches over all dependencies while running
  and restarts given target in case something was touched.

### pb CLI

```
pb [options] [processor:]target... [-- run args...]

Options:
  -j NUM        Maximum number of concurrent jobs
  --dry-run     Do not actually perform but print each action
  -e KEY=VAL    Passes environment variable to build environment.
  -r REL_TYPE   Release type (one of: release, debug, relwithdbg)
  -v            Print version and exit

Targets:
  all           Builds all targets
  clean         Cleans all targets
  install       Installs given targets
  TARGET_NAME   Builds given single target

Processors:
  build         Compiles given targets (default).
  deb           Creates .deb packages for given target on current platform.
  run           Executes given target. Optional cmd args can be passed.
  test          Runs tests for given target.
  autotool      Generates autoconf/automake/libtool target files.

Examples:

  # same as `pb build:all`
  pb all

  # builds libxzero in debug mode
  pb build:libxzero -r debug

  # builds libxzero and runs all tests for libxzero
  pb test:libxzero

  # builds all dependencies and then runs given target
  pb run:examples/http-hello1

  # generates all .deb files for the current operating system (ie. Ubuntu 14.04)
  pb deb:all -r release
```

### .pb File Examples

```
# libxzero/libxzero.pb
package "libxero-dev"

# install some headers
target "xzero" as fs {
  src [
    include/xzero/HttpChannel.h
    include/xzero/HttpRequest.h
    include/xzero/HttpResponse.h
  ]
  src with "m4" [
    include/xzero/sysconfig.h
  ]

  install_prefix "include/xzero"
}

target "pkgconfig" as fs {
  # files should be processed with m4
  src with "m4" [
    libxzero.pc
  ]

  # can contain variables, such as @PREFIX@ etc.
  install_prefix "lib/pkgconfig"
}
```

```
# libxzero/libxzero.pb
package "libxzero"

# hard dependencies
depends [
  ":pthread"
  "pkgconfig:ev"
]

# soft/optional dependencies
uses [
  # uses pkgconfig to detect the package
  "pkgconfig:zlib"
]

target "xzero" as library {
  src [
    "src/http/HttpChannel.cc"
    "src/http/HttpRequest.cc"
    "src/http/HttpResponse.cc"
    "src/http/v1/HttpConnection.cc"
    "src/http/mock/MockConnection.cc"
  ]

  # links against the given (prior) declared external packages
  link [
    "pthread"
    "ev"
    "zlib"
  ]

  test [
    "test/http/HttpChannel-test.cc"
    "test/http/Http1-test.cc"
  ]
  test_link [
    "gtest"
  ]
}

```

### A Note on C/C++

* Variables CXX, CFLAGS, CXXFLAGS, LDFLAGS etc will be honored.
* When those env vars changed since previous run, the target gets rebuilt too.






