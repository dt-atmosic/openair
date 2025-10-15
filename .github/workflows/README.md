# OpenAir GitHub Actions Workflows

This directory contains GitHub Actions workflows for building and testing OpenAir applications.

## Build Applications Workflow

The `build-applications.yml` workflow automatically discovers and builds OpenAir applications and samples using `west build --sysbuild`, creating `.atm` programming archives and publishing them as GitHub releases.

### Automatic Test Discovery and Efficient Building

The workflow is designed for efficiency and scalability:

1. **Discovers applications**: Finds all `sample.yaml` and `testcase.yaml` files in the repository
2. **Groups by application directory**: Creates one job per application directory (`app_dir`)
3. **Builds all variants**: Each job builds all board/test combinations for that application
4. **Creates archives**: Bundles all `.atm` files for an application into a single `.tar.gz` archive
5. **Publishes to releases**: Uploads archives to GitHub releases (avoids 1000-file limit)

**Key benefits:**
- **Efficient**: Sets up the build environment once per application, then builds all variants
- **Scalable**: Avoids GitHub's 256 matrix configuration limit by grouping builds
- **Resilient**: Individual build failures don't stop other builds (fail-fast: false)
- **Organized**: Archives are named by application directory, containing all board/test variants

This means you don't need to manually update the workflow when adding new applications or tests - just add a `sample.yaml` or `testcase.yaml` file with appropriate test names and they'll be automatically built.

### Current Configuration

- **Runner**: Ubuntu 22.04
- **Board Discovery**: Automatically discovers boards from `board.yml` files under `boards/atmosic/`
  - Excludes `atmevk-02` directory
  - Uses `//ns` variant for all boards
  - Discovers both ATM33 and ATM34 series boards
- **Test Discovery**: Automatic from `sample.yaml` and `testcase.yaml` files
- **Build Options**: Creates `.atm` programming archives with `-DSB_CONFIG_ATM_ARCH=y -DSB_CONFIG_ATM_ARCH_ERASE_ALL=y`
- **Output**: Archives of `.atm` files are uploaded to GitHub releases

### Workflow Triggers

- Pull requests to `main` branch
- Pushes to `main` branch
- Manual workflow dispatch (via GitHub Actions UI)

### Workflow Jobs

#### 1. Discovery Job (`discover-applications`)

This job discovers all applications and their test configurations:

1. **Checkout repository**: Gets the source code
2. **Install PyYAML**: Installs Python YAML parser
3. **Discover boards and tests**: Runs Python script to:
   - Discover boards from `board.yml` files under `boards/atmosic/` (excluding `atmevk-02`)
   - Walk through all directories to find `sample.yaml` and `testcase.yaml` files
   - Parse the `tests:` section from each file
   - **Group by `app_dir`**: Creates one matrix entry per application directory
   - Each entry contains all board/test combinations for that application
4. **Output matrix**: Provides the matrix to the build jobs

#### 2. Build Jobs (`build`)

One job runs per application directory, building all its variants:

1. **System Dependencies**: Installs required Ubuntu packages for Zephyr development
2. **Python Environment**: Sets up Python and installs `west` build tool
3. **Zephyr SDK**: Downloads and installs Zephyr SDK 0.16.8 with ARM toolchain (cached)
4. **West Workspace**: Initializes the west workspace and fetches dependencies (cached)
5. **Python Dependencies**: Installs required Python packages
6. **Build all configurations**: Loops through all board/test combinations:
   - Runs `west build --sysbuild` for each configuration
   - Includes `-DSB_CONFIG_ATM_ARCH=y -DSB_CONFIG_ATM_ARCH_ERASE_ALL=y` to create `.atm` programming archives
   - Continues on individual build failures (doesn't stop the job)
   - Reports success/failure count at the end
7. **Create archive**: Bundles all `.atm` files into a `.tar.gz` archive named after the `app_dir`
8. **Upload to Release**: Uploads the archive to GitHub release (tagged by PR number or build number)
9. **Summary**: Generates a build summary with all generated `.atm` files and archive info

### Caching

The workflow uses GitHub Actions caching to speed up subsequent builds:

- **Zephyr SDK**: Cached to avoid re-downloading (~500MB)
- **West Modules**: Cached based on `west.yml` hash to avoid re-fetching dependencies

### Release Management

The workflow automatically creates GitHub releases with `.atm` programming archive bundles:

- **For Pull Requests**: Creates a prerelease tagged as `pr-{number}-{run_number}`
- **For Main Branch**: Creates a release tagged as `build-{run_number}`
- **Archive Naming**: Each archive is named `{app_dir}.tar.gz` (with `/` replaced by `-`)
- **Archive Contents**: Contains all `.atm` files for all board/test combinations of that application
- **Automatic Upload**: All archives from successful builds are uploaded to the release
- **Scalability**: Avoids GitHub's 1000-file-per-release limit by bundling files into archives

### Adding New Tests

To add a new test configuration that will be automatically built:

1. **Create or update a test definition file** in your application/sample directory:
   - Use `sample.yaml` for samples
   - Use `testcase.yaml` for test cases

2. **Add test configurations**:

```yaml
sample:
  name: My Application
  description: Description of my application
tests:
  my_app.test_variant:
    sysbuild: true
    tags: my_tag atm33
    extra_args:
      - SB_CONFIG_SPE=y
  my_app.other_variant:
    sysbuild: true
```

3. **Commit and push** - the workflow will automatically discover and build your new tests on all configured boards

### Adding New Boards

Boards are automatically discovered from `board.yml` files under `boards/atmosic/`. To add a new board:

1. Create a new board directory under `boards/atmosic/` (e.g., `boards/atmosic/atm35evk/`)
2. Add a `board.yml` file with the board definition following the Zephyr board format
3. Commit and push - the workflow will automatically discover and build all tests on the new board

**Note**:
- The workflow currently uses the `//ns` variant for all boards (hard-coded)
- The `atmevk-02` directory is excluded from board discovery
- All discovered boards will build all discovered tests

### Example Test Definition

Here's a complete example of a `sample.yaml` file:

```yaml
sample:
  name: Hello World
  description: Simple hello world application
common:
  sysbuild: true
  tags: introduction
tests:
  samples.hello_world:
    tags: introduction atm33 atm34
    extra_args:
      - SB_CONFIG_SPE=y
  samples.hello_world.mcuboot:
    tags: introduction atm33 atm34 mcuboot
    extra_args:
      - SB_CONFIG_SPE=y
      - SB_CONFIG_BOOTLOADER_MCUBOOT=y
```

Both test configurations will be automatically discovered and built on all configured boards.

### Current Features

The workflow currently includes:

1. ✅ **Automatic Test Discovery**: Discovers all tests from `sample.yaml` and `testcase.yaml` files
2. ✅ **Multi-Board Support**: Builds tests on multiple board variants (ATM33 and ATM34)
3. ✅ **Efficient Grouping**: Groups builds by application directory to minimize setup overhead
4. ✅ **Programming Archives**: Creates `.atm` programming archives for easy device programming
5. ✅ **Archive Bundling**: Bundles all `.atm` files per application into `.tar.gz` archives
6. ✅ **GitHub Releases**: Automatically uploads archives to GitHub releases
7. ✅ **Build Summaries**: Generates per-application build summaries with archive contents
8. ✅ **Caching**: Caches SDK and west modules for faster builds
9. ✅ **Fail-Fast Disabled**: Continues building other tests even if one fails

