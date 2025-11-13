# Changelog

All notable changes to @system/bsp will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2025-11-13

### Added

- Initial release as standalone package
- Migrated from monolithic pkgs repository
- Added comprehensive documentation (README.md, CLAUDE.md)
- Added proper dependency declarations in package.conf
- Declared external command requirements: bspc, systemctl

### Changed

- Reorganized as part of pkgs ecosystem Phase 3 (Package Organization)
- Updated package structure to follow pkgs standards

### Migration Notes

This package was previously part of the monolithic `~/.pkgs` repository. It has been extracted into its own repository as part of the pkgs ecosystem redesign.

For users upgrading from the old system:
- No breaking changes to functionality
- Installation method changed from manual stow to `pkg-cli install`
- Package naming now includes collection scope: `@system/bsp`

---

**Legend**:
- **Added**: New features
- **Changed**: Changes in existing functionality
- **Deprecated**: Soon-to-be removed features
- **Removed**: Removed features
- **Fixed**: Bug fixes
- **Security**: Security fixes
