# ci-cache-cmake-build

## 1. Create workflow to create reference build for each develop branch commit

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
      shell: bash
      run: |
        cmake -S codedir -B build # some CMake args
        cmake --build build --parallel 2
        cd codedir
        git ls-files | while read -r file; do
          # Skip files in the build directory
          if [[ "$file" != "${{ inputs.build-dir }}/"* && -f "$file" ]]; then
            # Calculate file hash
            file_hash=$(git hash-object "$file")
            # Store in manifest with pipe separator
            echo "$file|$file_hash" >> reference-manifest.txt
          fi
        done
        echo "DEVELOP_HASH=$(git rev-parse HEAD)" >> $GITHUB_ENV

    - name: Cache save
      uses: actions/cache/save@v4
      with:
        path: |
          build/
          reference-manifest.txt
        key: cmake-build-cache-AlexanderRichert-NOAA-${{ env.DEVELOP_HASH }}
```

## 2. Add ci-cache-cmake-build action to regular CI workflows in place of CMake build

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
