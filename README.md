.github/workflows/build.yml

name: Build AntForestPort dylibs

on:
  workflow_dispatch:
  push:
    branches: [ main ]

permissions:
  contents: read

jobs:
  build:
    runs-on: macos-15
    timeout-minutes: 30

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Prepare source
        run: |
          set -euo pipefail
          archive="$(find . -maxdepth 1 -type f -name '*.zip' | head -n 1)"
          if [ -z "$archive" ]; then
            echo "Source ZIP not found"
            exit 1
          fi
          mkdir -p source
          ditto -x -k "$archive" source
          makefile="$(find source -type f -name Makefile | head -n 1)"
          if [ -z "$makefile" ]; then
            echo "Makefile not found"
            exit 1
          fi
          echo "SOURCE_DIR=$(dirname "$makefile")" >> "$GITHUB_ENV"

      - name: Select Xcode
        run: |
          sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
          xcodebuild -version
          xcrun --sdk iphoneos --show-sdk-version

      - name: Run tests
        working-directory: ${{ env.SOURCE_DIR }}
        run: make test

      - name: Build dylibs
        working-directory: ${{ env.SOURCE_DIR }}
        run: |
          make clean
          make

      - name: Verify outputs
        working-directory: ${{ env.SOURCE_DIR }}
        run: |
          set -euo pipefail
          test -s build/AntForestPort-v3.2.7.dylib
          test -s build/AntForestPort-v3.2.7-iOS14.dylib
          xcrun lipo -info build/AntForestPort-v3.2.7.dylib
          xcrun lipo -info build/AntForestPort-v3.2.7-iOS14.dylib

      - name: Package outputs
        working-directory: ${{ env.SOURCE_DIR }}
        run: |
          mkdir -p "$GITHUB_WORKSPACE/artifacts"
          cp build/*.dylib "$GITHUB_WORKSPACE/artifacts/"
          shasum -a 256 "$GITHUB_WORKSPACE"/artifacts/* > "$GITHUB_WORKSPACE/artifacts/SHA256SUMS.txt"

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: AntForestPort-ios-dylibs
          path: artifacts/*
          retention-days: 14
          if-no-files-found: error
