# ci-cache-cmake-build

This is an experimental custom GitHub action for caching CMake builds in order to enable partial, minimal rebuilds, rebuilding only the files that have changed compared with the source code used in the reference (cached) build.
