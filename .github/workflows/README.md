# OpenAir GitHub Actions Workflows

This directory contains GitHub Actions workflows for building and testing OpenAir applications.

## Build Applications Workflow

The `build-applications.yml` workflow builds OpenAir applications across various Atmosic boards using `west build --sysbuild`.

This workflow follows **Option 1** from the [OpenAir installation guide](https://atmosic.com/public/OpenAir_SDK_doc/getting_started_guide/installation.html) - using the standard Zephyr development environment setup with the Zephyr SDK.

### Current Configuration

- **Runner**: Ubuntu 22.04
- **Board**: ATMEVK-3330e-QN-7//ns (ATM33 series)
- **Application**: samples/sysbuild/hello_world
- **Build Configuration**: samples.sysbuild.hello_world.atm (with SPE)

### Workflow Triggers

- Pull requests to `main` branch
- Pushes to `main` branch
- Manual workflow dispatch (via GitHub Actions UI)

### Setup Steps

1. **System Dependencies**: Installs required Ubuntu packages for Zephyr development
2. **Python Environment**: Sets up Python and installs `west` build tool
3. **Zephyr SDK**: Downloads and installs Zephyr SDK 0.16.8 with ARM toolchain
4. **West Workspace**: Initializes the west workspace and fetches dependencies
5. **Python Dependencies**: Installs required Python packages
6. **Build**: Builds the application using west with sysbuild
7. **Summary**: Generates a build summary with artifact information

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

### Expanding the Build Matrix

To build multiple applications and boards, you can add a matrix strategy. Example:

```yaml
jobs:
  build:
    runs-on: ubuntu-22.04
    strategy:
      fail-fast: false
      matrix:
        include:
          - board: ATMEVK-3330e-QN-7//ns
            app: samples/sysbuild/hello_world
            test: samples.sysbuild.hello_world.atm
          - board: ATMEVK-3405-PQK-5//ns
            app: applications/fp_tag
            test: applications.fp_tag.atm
          # Add more combinations here
```

Then update the build step to use matrix variables:
```yaml
- name: Build application
  run: |
    cd ${{ github.workspace }}
    west build -p always -b ${{ matrix.board }} openair/${{ matrix.app }} --sysbuild -T ${{ matrix.test }}
```

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

Each application has multiple test configurations defined in its `sample.yaml` file:

- Basic builds with SPE (Secure Processing Environment)
- Builds with MCUboot bootloader
- Builds with different ATMWSTK configurations
- Builds with various DFU options

Refer to the `sample.yaml` file in each application directory for available test configurations.

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

### Future Enhancements

Potential improvements to consider:

1. **Artifact Upload**: Upload build artifacts (hex, bin, elf files) for download
2. **Matrix Builds**: Build multiple applications and boards in parallel
3. **Selective Building**: Only build applications affected by PR changes
4. **Build Time Tracking**: Report and track build times
5. **Binary Size Comparison**: Compare binary sizes across builds
6. **Scheduled Builds**: Add nightly builds for comprehensive testing
7. **Status Badges**: Add build status badges to README

