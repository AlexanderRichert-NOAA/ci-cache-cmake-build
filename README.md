# ci-cache-cmake-build

This repository provides custom GitHub actions for caching CMake builds in
order to speed up future rebuilds in, say, workflows triggered by pull
requests.

The `cache-reference-build` action, described in detail below, builds the CMake
project for the repository's authoritative branch (e.g., develop) and caches
the build directory. The `build-code` then uses that cached build directory as
a starting point so that only modified files/targets need to be rebuilt. If no
corresponding cache entry exists for the authoritative repository's develop
branch, `build-code` will build the code normally, i.e., from scratch.

## 1. Run `cache-reference-build` on develop branch commit

Create a workflow called, say, cache-reference-build.yml, which will look
something like the following, and will run whenever a commit is merged into
develop:
```yaml
name: 'Cache reference build from merged commit on develop'
on:
  push:
    branches:
      - 'develop'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: Checkout code
      uses: actions/checkout@v4
      with:
        path: codedir

    - name: Generate reference build
      uses: AlexanderRichert-NOAA/ci-cache-cmake-build/cache-reference-build@main
      with:
        source-dir: codedir
        build-dir: build
        config-args: -DOPENMP=OFF
```

### `cache-reference-build` inputs

| Name | Description | Default | Required |
| ---- | ----------- | ------- | -------- |
| `source-dir` | Source code directory, relative to github.workspace | `codedir` | No |
| `build-dir` | CMake build directory, relative to github.workspace | `build` | No |
| `config-args` | Additional config arguments (passed to initial CMake invocation) | _empty string_ | No |
| `build-args` | Additional build arguments (passed to build tool like make or ninja) | _empty string_ | No |
| `cache-key-prefix` | Cache key prefix | `cmake-build-cache` | No |
| `build-system` | Build "Unix Makefiles" or "Ninja" | `Ninja` | No |


## 2. Run `build-code` action in regular CI workflows in place of CMake build

Using the custom action might look something like the following. Note that on a
cache miss, it will behave normally, i.e., rebuild from scratch, without
further intervention.
```yaml
name: Build and Test
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-24.04

    steps:

    - name: Installs
      run: sudo apt-get install doxygen graphviz 

    - name: checkout
      uses: actions/checkout@v4
      with: 
        path: codedir

    - name: Build
      uses: AlexanderRichert-NOAA/ci-cache-cmake-build/build-code@main
      with:
        source-dir: codedir
        build-dir: build/
        config-args: -DOPENMP=OFF # Use the same CMake args as in the reference build above
    
    - name: test
      run: ctest --test-dir build --output-on-failure
```

### `build-code` inputs

| Name | Description | Default | Required |
| ---- | ----------- | ------- | -------- |
| `source-dir` | Source code directory, relative to github.workspace (must be same as reference source dir name so CMake can use it) | `.` | No |
| `build-dir` | CMake build directory, relative to github.workspace | `build` | No |
| `cache-key-prefix` | Cache key prefix | `cmake-build-cache` | No |
| `config-args` | Additional config arguments (passed to initial CMake invocation) | _empty string_ | No |
| `build-args` | Additional build arguments (passed to build tool like make or ninja) | _empty string_ | No |
| `force-reconfigure` | Force CMake reconfiguration even if cache exists | `false` | No |
| `parent-org` | GitHub org with reference repo/branch | `NOAA-EMC` | No |
| `reference-branch` | Name of reference branch | `develop` | No |
| `build-system` | Build "Unix Makefiles" or "Ninja" | `Ninja` | No |
