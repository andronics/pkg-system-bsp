# Modern Stow Package Manager

**Package**: `@system/bsp`
**Version**: 0.1.0
**Repository**: `pkg-system-bsp`

## Overview

Modern Stow Package Manager

## Installation

### Prerequisites

This package requires the following external commands:

- `bspc`
- `systemctl`

### Using pkg-cli

```bash
# Install from GitHub
pkg-cli install @system/bsp

# Or install from local source
cd ~/.pkgs
stow bsp
```

## Usage

### System Integration

This package provides system integration for modern stow package manager.


## Configuration

Configuration files are typically stored in:
- `~/.config/bsp/` - User configuration
- `~/.local/share/bsp/` - Application data

## Uninstallation

```bash
# Using pkg-cli
pkg-cli uninstall @system/bsp

# Or manual unstow
cd ~/.pkgs
stow -D bsp
```

## Development

See [CLAUDE.md](CLAUDE.md) for development guidelines and AI assistance instructions.

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/andronics/pkg-system-bsp/issues)
- **Discussions**: [GitHub Discussions](https://github.com/andronics/pkg-system-bsp/discussions)
- **Documentation**: [pkgs ecosystem docs](https://github.com/andronics/.pkgs)

## Version History

See [CHANGELOG.md](CHANGELOG.md) for version history.
