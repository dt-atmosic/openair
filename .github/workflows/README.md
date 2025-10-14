# OpenAir GitHub Actions Workflows

This directory contains GitHub Actions workflows for building and testing OpenAir applications.

## Build Applications Workflow

The `build-applications.yml` workflow automatically discovers and builds OpenAir applications and samples using `west build --sysbuild`, creating `.atm` programming archives and publishing them as GitHub releases.

### Automatic Test Discovery

The workflow automatically discovers all test configurations to build:

1. **Searches for test definition files**: Finds all `sample.yaml` and `testcase.yaml` files in the repository
2. **Parses test configurations**: Extracts test names from the `tests:` section of each file
3. **Filters tests by board**: Each board has configurable test name suffixes (e.g., `.atm`)
4. **Creates matrix for multiple boards**: Builds each matching test on all configured boards
5. **Builds in parallel**: Uses a GitHub Actions matrix to build all discovered tests concurrently

This means you don't need to manually update the workflow when adding new applications or tests - just add a `sample.yaml` or `testcase.yaml` file with appropriate test names and they'll be automatically built.

### Current Configuration

- **Runner**: Ubuntu 22.04
- **Boards** (with test suffixes):
  - **ATM33 series** (6 boards):
    - `ATMEVK-3330-QN-6//ns`: Tests ending with `.atm`
    - `ATMEVK-3330e-QN-6//ns`: Tests ending with `.atm`
    - `ATMEVK-3330e-QN-7//ns`: Tests ending with `.atm`
    - `ATMEVK-3325-CM-6//ns`: Tests ending with `.atm`
    - `ATMEVK-3325-QK-6//ns`: Tests ending with `.atm`
    - `ATMEVK-3325-LQK-6//ns`: Tests ending with `.atm`
  - **ATM34 series** (6 boards):
    - `ATMEVK-3405-PQK-5//ns`: Tests ending with `.atm`
    - `ATMEVK-3425-YQK-5//ns`: Tests ending with `.atm`
    - `ATMEVK-3430e-YQN-5//ns`: Tests ending with `.atm`
    - `ATMEVK-3405-YBV-5//ns`: Tests ending with `.atm`
    - `ATMBTCSTAG-3405//ns`: Tests ending with `.atm`
    - `ATMEVK-3405-WQK-5//ns`: Tests ending with `.atm`
- **Discovery**: Automatic from `sample.yaml` and `testcase.yaml` files
- **Build Options**: Creates `.atm` programming archives with `-DSB_CONFIG_ATM_ARCH=y -DSB_CONFIG_ATM_ARCH_ERASE_ALL=y`
- **Output**: `.atm` files are uploaded to GitHub releases

### Workflow Triggers

- Pull requests to `main` branch
- Pushes to `main` branch
- Manual workflow dispatch (via GitHub Actions UI)

### Workflow Jobs

#### 1. Discovery Job (`discover-applications`)

This job runs first and discovers all test configurations:

1. **Checkout repository**: Gets the source code
2. **Install PyYAML**: Installs Python YAML parser
3. **Discover tests**: Runs Python script to:
   - Walk through all directories
   - Find `sample.yaml` and `testcase.yaml` files
   - Parse the `tests:` section from each file
   - Filter tests based on board-specific suffixes
   - Create matrix entries for each test × board combination
   - Output a JSON matrix of all discovered test/board combinations

#### 2. Build Job (`build`)

This job runs in parallel for each discovered test/board combination:

1. **System Dependencies**: Installs required Ubuntu packages for Zephyr development
2. **Python Environment**: Sets up Python and installs `west` build tool
3. **Zephyr SDK**: Downloads and installs Zephyr SDK 0.16.8 with ARM toolchain (cached)
4. **West Workspace**: Initializes the west workspace and fetches dependencies (cached)
5. **Python Dependencies**: Installs required Python packages
6. **Build**: Builds the specific test configuration on the specific board using west with sysbuild
   - Includes `-DSB_CONFIG_ATM_ARCH=y -DSB_CONFIG_ATM_ARCH_ERASE_ALL=y` to create `.atm` programming archives
7. **Find .atm files**: Locates all generated `.atm` programming archives
8. **Upload to Release**: Uploads `.atm` files to a GitHub release (tagged by PR number or build number)
9. **Summary**: Generates a build summary with `.atm` file information

### Caching

The workflow uses GitHub Actions caching to speed up subsequent builds:

- **Zephyr SDK**: Cached to avoid re-downloading (~500MB)
- **West Modules**: Cached based on `west.yml` hash to avoid re-fetching dependencies

### Release Management

The workflow automatically creates GitHub releases with `.atm` programming archives:

- **For Pull Requests**: Creates a prerelease tagged as `pr-{number}-{run_number}`
- **For Main Branch**: Creates a release tagged as `build-{run_number}`
- **File Naming**: Each `.atm` file is named with the pattern `{test_name}-{board}-{filename}.atm`
- **Automatic Upload**: All `.atm` files from successful builds are uploaded to the release

### Adding New Tests

To add a new test configuration that will be automatically built:

1. **Create or update a test definition file** in your application/sample directory:
   - Use `sample.yaml` for samples
   - Use `testcase.yaml` for test cases

2. **Add test configurations** with names matching the board suffixes (e.g., ending in `.atm`):

```yaml
sample:
  name: My Application
  description: Description of my application
tests:
  my_app.test_variant.atm:  # Will be discovered and built on boards with .atm suffix
    sysbuild: true
    tags: my_tag atm33
    extra_args:
      - SB_CONFIG_SPE=y
  my_app.other_variant:  # Will NOT be built (doesn't match any board suffix)
    sysbuild: true
```

3. **Commit and push** - the workflow will automatically discover and build your new test on the next run

### Adding New Boards

To add a new board to the build matrix:

1. Edit `.github/workflows/build-applications.yml`
2. Add the board to the `boards` dictionary with its test suffixes:

```python
boards = {
    # ATM33 series boards
    'ATMEVK-3330-QN-6//ns': ['.atm'],
    'ATMEVK-3330e-QN-6//ns': ['.atm'],
    # ... other boards ...
    'NEW-BOARD-NAME//ns': ['.atm', '.custom']  # Can have multiple suffixes
}
```

3. Commit and push - tests matching the board's suffixes will be built on that board

**Note**: The current configuration builds all 12 boards (6 ATM33 + 6 ATM34) with the `//ns` variant and `.atm` test suffix.

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
  samples.hello_world.atm:
    tags: introduction atm33 atm34
    extra_args:
      - SB_CONFIG_SPE=y
  samples.hello_world.atm.mcuboot:
    tags: introduction atm33 atm34 mcuboot
    extra_args:
      - SB_CONFIG_SPE=y
      - SB_CONFIG_BOOTLOADER_MCUBOOT=y
```

Both test configurations will be automatically discovered and built because they end with `.atm`.

### Current Features

The workflow currently includes:

1. ✅ **Automatic Test Discovery**: Discovers all tests from `sample.yaml` and `testcase.yaml` files
2. ✅ **Configurable Board Filters**: Each board can specify which test name suffixes to build
3. ✅ **Multi-Board Support**: Builds tests on multiple board variants (ATM33 and ATM34)
4. ✅ **Matrix Builds**: Builds multiple applications and tests in parallel
5. ✅ **Programming Archives**: Creates `.atm` programming archives for easy device programming
6. ✅ **GitHub Releases**: Automatically uploads `.atm` files to GitHub releases
7. ✅ **Build Summaries**: Generates per-test build summaries with `.atm` file information
8. ✅ **Caching**: Caches SDK and west modules for faster builds
9. ✅ **Fail-Fast Disabled**: Continues building other tests even if one fails

