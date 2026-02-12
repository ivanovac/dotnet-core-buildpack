# Adding cflinuxfs5 Stack Support to Cloud Foundry .Net Core Buildpacks

## Overview
Successfully added cflinuxfs5 stack support to three Cloud Foundry buildpacks: .NET Core, Go, and Staticfile. All integration tests now pass.

## .NET Core Buildpack Changes
1. Built Dependencies (12 total)
Location: output-dotnet and output-dotnet-libs

- 10 .NET packages (~867 MB):
  - dotnet-sdk: 8.0.415, 9.0.306, 10.0.102
  - dotnet-runtime: 8.0.21, 9.0.10, 10.0.2
  - dotnet-aspnetcore: 8.0.21, 9.0.10, 10.0.2
  - bower: 1.8.14
    
- 2 Native libraries (~2.9 MB):
  - libunwind 1.8.3
  - libgdiplus 6.1


**Key Build Fix:** Native library tarballs must have files at root level:
```
# Correct:
tar -czf "$OUTPUT_FILE" -C "$PKG_DIR" .
# Wrong (creates wrapper directory):
tar -czf "$OUTPUT_FILE" "$PKG_DIR"
```

2. Script Modifications

install_go.sh (Line 8):

```
# Before:
if [[ "${CF_STACK:-}" != "cflinuxfs3" && "${CF_STACK:-}" != "cflinuxfs4" ]]; then

# After:
if [[ "${CF_STACK:-}" != "cflinuxfs3" && "${CF_STACK:-}" != "cflinuxfs4" && "${CF_STACK:-}" != "cflinuxfs5" ]]; then
```

package.sh (Added export in stack != "any" block):

```
if [[ "${stack}" != "any" ]]; then
    export CF_STACK="${stack}"  # Added this line
    stack_flag="--stack=${stack}"
fi
```

3. Go Code Changes at finalize phaze

finalize.go (Line 17):

```
// Before:
var stackToRuntimeRID = map[string]string{
    "cflinuxfs3": "linux-x64",
    "cflinuxfs4": "linux-x64",
}

// After:
var stackToRuntimeRID = map[string]string{
    "cflinuxfs3": "linux-x64",
    "cflinuxfs4": "linux-x64",
    "cflinuxfs5": "linux-x64",  // Added
}
```

4. Manifest Updates
manifest.yml:
 - Added 12 cflinuxfs5 dependency entries with GitHub release URLs
 - All pointing to `v2.5.1-beta-cflinuxfs5` release
 - Included SHA256 checksums for each dependency
 - **Critical:** Added install_go.sh to `include_files` section

VERSION:
 - Updated to 2.5.1-beta


5. Integration Test Support
init_test.go:
Added environment variable overrides for local buildpack testing - 
Taking the go-buildpack path from the environment variable rather downloading it from the github repo. This way we can pass to the test environment go-buildpack that is built for the cflinuxfs5 stack:

```
func downloadBuildpack(name string) (string, error) {
    envVar := fmt.Sprintf("%s_BUILDPACK_FILE", strings.ToUpper(name))
    if envFile := os.Getenv(envVar); envFile != "" {
        if _, err := os.Stat(envFile); err == nil {
            return envFile, nil
        }
    }
    // ... rest of function
}
```

## Go Buildpack Changes

1. Script Modifications
install_go.sh: Added cflinuxfs5 to stack validation (same as .NET Core)

package.sh: Added CF_STACK export (same as .NET Core)

2. Manifest Updates
manifest.yml:

 - **Critical**: Added install_go.sh to include_files section

## Staticfile Buildpack Changes

1. Built Dependencies (2 nginx versions)
**Location**: output-staticfile

 - nginx 1.26.2 (~4.5 MB)
 - nginx 1.27.3 (~4.5 MB)

Built with modules: SSL, realip, gzip_static, HTTP/2, stub_status, auth_request

2. Script Modifications
install_go.sh: Added cflinuxfs5 to stack validation

package.sh: Added CF_STACK export

3. Manifest Updates
manifest.yml:

 - Added 2 nginx cflinuxfs5 entries with GitHub release URLs
 - Pointing to v1.6.34-cflinuxfs5 release
 - **Critical**: Added install_go.sh to include_files section

## Critical Discovery: Runtime Scripts Must Be Packaged

**Problem**: The supply and finalize wrapper scripts source install_go.sh at runtime, not during packaging. If this script isn't included in the buildpack package, the stack validation fails when the buildpack runs.

**Solution**: Added install_go.sh to the include_files section in manifest.yml for all three buildpacks.

## Build & Release Process
1. For Each Buildpack:
```
# Build dependencies (if needed)
docker run --rm -v $(pwd)/build-script.sh:/build.sh \
  -v $(pwd)/output:/output cloudfoundry/cflinuxfs5 bash /build.sh

# Upload to GitHub release
# (Upload .tgz files to appropriate release tag)

# Rebuild binaries (to include Go code changes)
bash scripts/build.sh

# Package buildpack
bash scripts/package.sh --version <version> --stack cflinuxfs5 --cached

# Commit and push
git add . && git commit -m "Add cflinuxfs5 stack support"
git push fork cflinuxfs5
```

2. GitHub Releases Created:

 - **dotnet-core-buildpack**: v2.5.1-beta-cflinuxfs5 (12 files, ~872 MB)
 - **staticfile-buildpack**: v1.6.34-cflinuxfs5 (2 files, ~9 MB)
 - **go-buildpack**: Uses existing cflinuxfs3 dependencies (stack-agnostic)


## Integration Test Execution

```
cd dotnet-core-buildpack
export BUILDPACK_FILE=$PWD/build/buildpack.zip
export GO_BUILDPACK_FILE=/path/to/go-buildpack/build/buildpack.zip
export STATICFILE_BUILDPACK_FILE=/path/to/staticfile-buildpack/build/buildpack.zip
export CF_STACK=cflinuxfs5

bash scripts/integration.sh --platform docker --github-token <token>
```

## Why package.sh Must Export CF_STACK="${stack}"
The Problem: The install_go.sh script performs stack validation at runtime, as already shown above
```
if [[ "${CF_STACK:-}" != "cflinuxfs3" && "${CF_STACK:-}" != "cflinuxfs4" && "${CF_STACK:-}" != "cflinuxfs5" ]]; then
    echo "       **ERROR** Unsupported stack"
    exit 1
fi
```

This script is sourced by the buildpack's wrapper scripts (`supply`, `finalize`, `compile`) when they run **inside the container** during application staging.

#### The Issue

When `buildpack-packager` builds the buildpack, it:

1. Downloads dependencies listed in `manifest.yml` based on the stack
2. May invoke pre-packaging scripts that need to know which stack is being built

However, `buildpack-packager` doesn't automatically pass the `--stack` parameter as an environment variable to child processes. The wrapper scripts need `CF_STACK` set when they run `install_go.sh`.

#### The Solution
By explicitly exporting `CF_STACK` in `package.sh`:

```
if [[ "${stack}" != "any" ]]; then
    export CF_STACK="${stack}"  # Makes it available to subprocesses
    stack_flag="--stack=${stack}"
fi
```

This ensures:
1. **During packaging**: Any pre-packaging scripts that source `install_go.sh` can validate the stack correctly
2. **At runtime**: When the buildpack runs in the container, `CF_STACK` environment variable is set by Cloud Foundry, but the packaging process needs to handle it explicitly for build-time operations

#### Why This Matters
Without the export:

 - install_go.sh sees CF_STACK as empty (${CF_STACK:-} evaluates to empty string)
 - The validation check fails: empty ≠ cflinuxfs3, cflinuxfs4, or cflinuxfs5
 - Build fails with "Unsupported stack" error even though the stack is valid

With the export:

 - `CF_STACK` is properly set in the environment for all child processes
 - `install_go.sh` can validate that the stack is supported
 - Buildpack packaging and execution succeed


## Key Lessons Learned
1. Tarball Structure Matters: Native libraries must have files at tarball root, not in a wrapper directory
2. Runtime Scripts Need Packaging: Scripts sourced by bin/ wrappers must be in include_files
3. Multiple Touchpoints: Stack support requires changes in scripts, Go code, and manifest
4. Rebuild Binaries: After Go code changes, must rebuild with build.sh before packaging
5. SHA256 Checksums: Must be exact; buildpack-packager validates during packaging
6. Dependency Chain: Integration tests use multiple buildpacks, all need cflinuxfs5 support
