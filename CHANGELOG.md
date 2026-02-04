# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-02-04

### Added

#### Documentation
- Comprehensive README.md with badges, quick start, and system requirements
- CONFIGURATION.md with complete configuration reference
- SECURITY.md with production security hardening guide
- TROUBLESHOOTING.md with detailed problem-solving guide
- CONTRIBUTING.md with contribution guidelines
- CHANGELOG.md for tracking version changes
- .claude/CLAUDE.md for Claude Code development guidance

#### CI/CD
- GitHub Actions workflow for automated testing
  - YAML validation with yamllint
  - Markdown linting
  - Docker Compose validation
  - Full integration testing (stack deployment, health checks, GELF log ingestion)
- .yamllint configuration for YAML linting standards
- .markdownlint.json for Markdown formatting standards

#### Features
- Docker Compose based ELK stack deployment (Elasticsearch 9.3.0, Logstash 9.3.0, Kibana 9.3.0)
- Automated Kibana dashboard provisioning with pre-configured searches:
  - Error Logs panel (filters for errors, failures, exceptions)
  - Workload Statistics panel (monitors workload processing)
  - All Recent Logs panel (complete log stream)
- GELF log collection over UDP port 12201
- Daily log indices (log-YYYY.MM.dd pattern)
- Optimized column layouts for efficient log viewing
- Persistent Elasticsearch data storage

#### Configuration
- Environment variable support for version customization
- Configurable Docker network name
- Memory settings for Elasticsearch (512MB) and Logstash (256MB)
- Single-node Elasticsearch deployment (suitable for development/small deployments)

### Fixed
- Added missing newline at end of kibana.yaml (YAML linting requirement)
- Fixed YAML bracket spacing in GitHub Actions workflow

### Infrastructure
- Enhanced .gitignore to exclude secrets, logs, backups, and temporary files
- Apache 2.0 license

### Documentation Highlights
- 5,380+ lines of comprehensive documentation
- Complete security hardening guide for production deployments
- 15+ troubleshooting scenarios with solutions
- Configuration examples for minimal, production, and high-availability setups
- Index Lifecycle Management setup guide
- Backup and restore procedures
- Network security and firewall configuration examples

[0.1.0]: https://github.com/YOUR-ORG/sz_kibana_dashboard/releases/tag/v0.1.0
