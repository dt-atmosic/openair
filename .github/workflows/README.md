# OpenAir GitHub Actions Workflows

This directory contains GitHub Actions workflows for building and testing OpenAir applications.

## Build Applications Workflow

The `build-applications.yml` workflow automatically discovers and builds OpenAir applications and samples using `west build --sysbuild`.

This workflow follows **Option 1** from the [OpenAir installation guide](https://atmosic.com/public/OpenAir_SDK_doc/getting_started_guide/installation.html) - using the standard Zephyr development environment setup with the Zephyr SDK.

### Automatic Test Discovery

The workflow automatically discovers all test configurations to build:

1. **Searches for test definition files**: Finds all `sample.yaml` and `testcase.yaml` files in the repository
2. **Parses test configurations**: Extracts test names from the `tests:` section of each file
3. **Filters for ATM tests**: Only includes tests with names ending in `.atm`
4. **Builds in parallel**: Uses a GitHub Actions matrix to build all discovered tests concurrently

This means you don't need to manually update the workflow when adding new applications or tests - just add a `sample.yaml` or `testcase.yaml` file with test names ending in `.atm` and they'll be automatically built.

### Current Configuration

- **Runner**: Ubuntu 22.04
- **Board**: ATMEVK-3330e-QN-7//ns (ATM33 series)
- **Discovery**: Automatic from `sample.yaml` and `testcase.yaml` files
- **Test Filter**: Only tests ending with `.atm`

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
   - Filter for test names ending with `.atm`
   - Output a JSON matrix of all discovered tests

#### 2. Build Job (`build`)

This job runs in parallel for each discovered test configuration:

1. **System Dependencies**: Installs required Ubuntu packages for Zephyr development
2. **Python Environment**: Sets up Python and installs `west` build tool
3. **Zephyr SDK**: Downloads and installs Zephyr SDK 0.16.8 with ARM toolchain (cached)
4. **West Workspace**: Initializes the west workspace and fetches dependencies (cached)
5. **Python Dependencies**: Installs required Python packages
6. **Build**: Builds the specific test configuration using west with sysbuild
7. **Upload Artifacts**: Uploads build artifacts (hex, bin, elf files)
8. **Summary**: Generates a build summary with artifact information

### Caching

The workflow uses GitHub Actions caching to speed up subsequent builds:

- **Zephyr SDK**: Cached to avoid re-downloading (~500MB)
- **West Modules**: Cached based on `west.yml` hash to avoid re-fetching dependencies

### Development Environment Setup

The workflow uses the standard Zephyr development environment setup:

1. **System Dependencies**: Installs all required packages for Zephyr development on Ubuntu 22.04
2. **Python & West**: Installs Python pip and the west meta-tool
3. **Zephyr SDK**: Downloads and installs the official Zephyr SDK with ARM toolchain
4. **West Workspace**: Initializes the workspace using the OpenAir repository as the manifest

This approach follows the official Zephyr Getting Started Guide and is suitable for CI/CD environments.

### Adding New Tests

To add a new test configuration that will be automatically built:

1. **Create or update a test definition file** in your application/sample directory:
   - Use `sample.yaml` for samples
   - Use `testcase.yaml` for test cases

2. **Add test configurations** with names ending in `.atm`:

```yaml
sample:
  name: My Application
  description: Description of my application
tests:
  my_app.test_variant.atm:  # Will be discovered and built
    sysbuild: true
    tags: my_tag atm33
    extra_args:
      - SB_CONFIG_SPE=y
  my_app.other_variant:  # Will NOT be built (doesn't end with .atm)
    sysbuild: true
```

3. **Commit and push** - the workflow will automatically discover and build your new test on the next run

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

### Available Applications

Based on the repository structure, the following applications can be built:

- `samples/sysbuild/hello_world` - Simple hello world with sysbuild
- `applications/combo_tag` - FMNA and FMDN Combo Tag
- `applications/fmna_tag` - Apple Find My Network Tag
- `applications/fp_tag` - Google Find My Device Network Tag
- `applications/sensor_beacon` - Sensor Beacon
- `applications/ss_fmna_tag` - Samsung Apple ComboTag
- `applications/ras_rreq_initiator` - RAS RREQ Initiator
- `applications/ras_rrsp_reflector` - RAS RRSP Reflector

### Available Boards

ATM33 series boards (examples):
- `ATMEVK-3330e-QN-7//ns`
- `ATMEVK-3330-QN-6//ns`
- `ATMEVK-3325-QK-6//ns`

ATM34 series boards (examples):
- `ATMEVK-3405-PQK-5//ns`
- `ATMEVK-3430e-YQN-5//ns`
- `ATMBTCSTAG-3405//ns`

See `boards/atmosic/` directory for the complete list of available boards.

### Build Configurations

Each application has multiple test configurations defined in its `sample.yaml` or `testcase.yaml` file:

- Basic builds with SPE (Secure Processing Environment)
- Builds with MCUboot bootloader
- Builds with different ATMWSTK configurations
- Builds with various DFU options

Only test configurations with names ending in `.atm` are automatically built by the CI workflow.

Refer to the `sample.yaml` or `testcase.yaml` file in each application directory for available test configurations.

### Troubleshooting

**Build fails with "board not found":**
- Verify the board name matches exactly (case-sensitive)
- Check that the board exists in `boards/atmosic/`
- Ensure the board variant (//ns, //no_TZ) is correct

**West update fails:**
- Check network connectivity
- Verify `west.yml` is valid
- Clear cache and retry

**Python dependency errors:**
- Ensure all requirements files are installed
- Check Python version compatibility (3.10+ recommended)

**SDK installation fails:**
- Verify SDK download URL is accessible
- Check available disk space
- Ensure all system dependencies are installed

### Current Features

The workflow currently includes:

1. ✅ **Automatic Test Discovery**: Discovers all tests from `sample.yaml` and `testcase.yaml` files
2. ✅ **Matrix Builds**: Builds multiple applications and tests in parallel
3. ✅ **Artifact Upload**: Uploads build artifacts (hex, bin, elf files) for download
4. ✅ **Build Summaries**: Generates per-test build summaries with artifact information
5. ✅ **Caching**: Caches SDK and west modules for faster builds
6. ✅ **Fail-Fast Disabled**: Continues building other tests even if one fails

### Future Enhancements

Potential improvements to consider:

1. **Multi-Board Support**: Build tests on multiple board variants
2. **Selective Building**: Only build applications affected by PR changes
3. **Build Time Tracking**: Report and track build times across runs
4. **Binary Size Comparison**: Compare binary sizes across builds
5. **Scheduled Builds**: Add nightly builds for comprehensive testing
6. **Status Badges**: Add build status badges to README
7. **Test Execution**: Run tests on hardware or in simulation

