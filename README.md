# Lebowski Opinions

Official opinion repository for [Lebowski](https://github.com/jgowdy/lebowski) - a Debian package builder with reproducible, opinionated builds.

## What are Opinions?

Opinions are YAML files that describe how to build Debian packages with specific modifications:
- Compilation flags (CFLAGS, CXXFLAGS, LDFLAGS)
- Configure options
- Build environment variables
- Custom toolchains (XSC, LLVM, etc.)
- Kernel configurations

Opinions enable **reproducible**, **transparent**, and **auditable** package customizations.

## Repository Structure

```
├── _base/              # Base opinions (templates for inheritance)
│   ├── xsc-baseline.yaml
│   └── xsc-v1.yaml
├── bash/              # Bash shell opinions
├── coreutils/         # GNU coreutils opinions
├── linux/             # Linux kernel opinions
├── nginx/             # nginx web server opinions
├── python3/           # Python interpreter opinions
└── ...                # More packages (49 total)
```

## Quick Start

### Using an Opinion

```bash
# Build with a specific opinion
lebowski build nginx --opinion-file nginx/performance.yaml

# Build from this repository
lebowski build bash \
  --opinion-file https://raw.githubusercontent.com/jgowdy/lebowski-opinions/main/bash/xsc.yaml
```

### Creating an Opinion

```yaml
# nginx/custom.yaml
version: "1.0"
package: nginx
opinion_name: custom
purity_level: pure-compilation
description: nginx optimized for my workload

modifications:
  cflags:
    - "-O3"
    - "-march=native"
    - "-flto"

  ldflags:
    - "-Wl,-O1"
    - "-Wl,--as-needed"
```

Test it:
```bash
lebowski build nginx --opinion-file nginx/custom.yaml
```

## Opinion Examples

### Standard Build (test-vanilla)
Most packages have a `test-vanilla` opinion for testing standard Debian builds:

```yaml
version: "1.0"
package: bash
opinion_name: test-vanilla
purity_level: pure-compilation
description: Standard bash build

modifications:
  env: {}
  cflags: []
  ldflags: []
```

### XSC Toolchain (Zero-Syscall)
XSC opinions use a custom toolchain that eliminates syscall instructions:

```yaml
# bash/xsc.yaml
extends: ../_base/xsc-baseline.yaml  # Inherits XSC toolchain setup
package: bash
opinion_name: xsc

modifications:
  configure_args:
    - "--enable-static-link"
    - "--without-bash-malloc"
```

The parent `xsc-baseline.yaml` provides:
- XSC container image (`lebowski/builder:xsc`)
- XSC toolchain paths and sysroot
- XSC runtime library (`-lxsc-rt`)

### Performance Optimizations

```yaml
# nginx/performance.yaml
package: nginx
opinion_name: performance
purity_level: configure-only

modifications:
  cflags:
    - "-O3"
    - "-march=native"
    - "-mtune=native"
    - "-flto"

  configure_args:
    - "--with-threads"
    - "--with-file-aio"
```

### Kernel Configuration

```yaml
# linux/gaming.yaml
package: linux
opinion_name: gaming
purity_level: configure-only

modifications:
  kernel_config:
    CONFIG_PREEMPT: "y"
    CONFIG_HZ_1000: "y"
    CONFIG_TCP_CONG_BBR: "m"
```

## Purity Levels

Opinions are categorized by **purity level** (trust level):

| Level | What's Allowed | Trust | Use Cases |
|-------|----------------|-------|-----------|
| `pure-compilation` | Only compiler flags (CFLAGS, LDFLAGS, defines) | **HIGHEST** | Performance tuning, optimization flags |
| `configure-only` | Build configuration options | **HIGH** | Feature flags, build-time options |
| `debian-patches` | Debian's official patches | **MEDIUM-HIGH** | Debian-maintained fixes |
| `upstream-patches` | Upstream project patches | **MEDIUM** | Backported fixes |
| `third-party-patches` | Community patches | **LOWER** | Experimental features |
| `custom` | Custom scripts and modifications | **LOWEST** | Advanced modifications |

**Best Practice**: Use the highest purity level possible. Lower levels require more justification.

## Toolchain Support

### Standard Toolchain
Default Debian GCC/G++ in `lebowski/builder:bookworm` container.

### XSC Toolchain (Zero-Syscall)
Custom toolchain for ring-based syscalls (no `syscall` instructions):

- **Container**: `lebowski/builder:xsc`
- **Base Opinion**: `_base/xsc-baseline.yaml`
- **Packages**: bash, coreutils, nginx, systemd, and more

See [XSC opinions](./_base/xsc-baseline.yaml) for details.

### Custom Toolchains
Opinions can specify any container image:

```yaml
container_image: "lebowski/builder:clang"  # Use LLVM/Clang
container_image: "lebowski/builder:aarch64"  # Cross-compile for ARM
```

## Opinion Inheritance

Opinions can extend parent opinions using `extends`:

```yaml
# bash/xsc.yaml
extends: ../_base/xsc-baseline.yaml  # Inherits XSC setup
package: bash
opinion_name: xsc

modifications:
  configure_args:
    - "--enable-static-link"  # Bash-specific additions
```

**Merge behavior**:
- Child overrides parent for single values
- Lists are concatenated (parent first, then child)
- Dictionaries are merged (child overrides parent keys)
- Conflicting compiler flags are normalized (child wins)

## Contributing

### Adding a New Opinion

1. **Fork and clone** this repository
2. **Create opinion file**: `<package>/<opinion-name>.yaml`
3. **Test the build**:
   ```bash
   lebowski build <package> --opinion-file <package>/<opinion-name>.yaml
   ```
4. **Verify output**: Check build artifacts and manifest
5. **Submit PR** with:
   - Opinion file
   - Build test results
   - Justification for purity level

### Opinion Schema

Required fields:
```yaml
version: "1.0"
package: <debian-package-name>
opinion_name: <unique-name>
purity_level: <one of the purity levels>
description: <what this opinion does>

modifications:
  # At least one modification
```

Optional fields:
```yaml
maintainer:
  name: Your Name
  email: you@example.com

tags: [performance, xsc, experimental]
debian_versions: [bookworm, bullseye]

extends: ../_base/parent-opinion.yaml

container_image: "lebowski/builder:custom"
```

### Validation

Before submitting:
```bash
# Validate YAML syntax
yamllint <package>/<opinion-name>.yaml

# Test build
lebowski build <package> --opinion-file <package>/<opinion-name>.yaml

# Verify purity level matches modifications
lebowski verify <output-dir>/*.lebowski-manifest.json
```

## Opinion Index

### System Packages
- **bash** - Bourne Again SHell (vanilla, xsc)
- **coreutils** - GNU core utilities (vanilla, xsc)
- **systemd** - System and service manager (xsc)
- **util-linux** - Miscellaneous system utilities (vanilla, xsc)

### Servers
- **nginx** - HTTP server (http3, performance, xsc)
- **postgresql** - SQL database (xsc)
- **redis** - Key-value store (performance)

### Development
- **python3** - Python interpreter (optimized, xsc-optimized)
- **git** - Version control (vanilla)
- **make** - Build automation (vanilla)

### Compression
- **gzip**, **bzip2**, **xz-utils**, **zstd** - Compression tools (vanilla)

### Kernel
- **linux** - Linux kernel (desktop-1000hz, gaming, server)

[See all opinions](.)

## Resources

- [Lebowski Documentation](https://github.com/jgowdy/lebowski/blob/main/README.md)
- [Opinion Format Specification](https://github.com/jgowdy/lebowski/blob/main/docs/opinion-format.md)
- [XSC Toolchain Details](https://github.com/jgowdy/lebowski/blob/main/docs/TOOLCHAINS.md)
- [Data-Driven Toolchains](https://github.com/jgowdy/lebowski/blob/main/docs/DATA-DRIVEN-TOOLCHAINS.md)

## License

MIT License - see [LICENSE](LICENSE) file.

## Questions?

- Open an issue on [Lebowski](https://github.com/jgowdy/lebowski/issues)
- Submit PR to this repository
- Join discussions in the Lebowski community
