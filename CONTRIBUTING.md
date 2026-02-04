# Contributing to Senzing ELK Stack

Thank you for your interest in contributing! This document provides guidelines for contributing to the Senzing ELK Stack deployment project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Reporting Bugs](#reporting-bugs)
- [Suggesting Enhancements](#suggesting-enhancements)
- [Development Process](#development-process)
- [Pull Request Process](#pull-request-process)
- [Testing Requirements](#testing-requirements)
- [Documentation Standards](#documentation-standards)
- [Code Review Process](#code-review-process)

## Code of Conduct

This project adheres to professional standards of conduct. Contributors are expected to:

- Be respectful and inclusive
- Accept constructive criticism gracefully
- Focus on what is best for the community
- Show empathy towards other community members

## How Can I Contribute?

### Reporting Bugs

Before submitting a bug report:

1. **Check existing issues** to avoid duplicates
2. **Verify the bug** by testing with the latest version
3. **Collect debug information** (see TROUBLESHOOTING.md)

**Submit a bug report:**

Create an issue with the following information:

```markdown
## Bug Description
[Clear description of the bug]

## Steps to Reproduce
1. Deploy with `docker compose -f kibana.yaml up -d`
2. Access Kibana at http://localhost:5601
3. Navigate to...
4. See error

## Expected Behavior
[What you expected to happen]

## Actual Behavior
[What actually happened]

## Environment
- OS: [e.g., Ubuntu 22.04]
- Docker version: [e.g., 24.0.0]
- Docker Compose version: [e.g., 2.20.0]
- ELK Stack versions: [from kibana.yaml]

## Logs
```
[Paste relevant logs from `docker compose -f kibana.yaml logs`]
```

## Additional Context
[Any other relevant information]
```

### Suggesting Enhancements

Enhancement suggestions are welcome! Please include:

1. **Use case**: Describe the problem you're trying to solve
2. **Proposed solution**: How you envision it working
3. **Alternatives considered**: Other approaches you've thought about
4. **Impact**: Who would benefit from this enhancement

**Template:**

```markdown
## Enhancement Description
[Clear description of the enhancement]

## Use Case
[Describe the problem this solves]

## Proposed Solution
[How it should work]

## Alternatives Considered
[Other approaches you've considered]

## Benefits
- [List benefits]

## Implementation Suggestions
[If you have ideas on how to implement]
```

### Improving Documentation

Documentation improvements are always appreciated:

- Fix typos or clarify confusing sections
- Add examples for common use cases
- Improve troubleshooting guides
- Translate documentation (if applicable)
- Add diagrams or visualizations

## Development Process

### Setting Up Development Environment

1. **Fork the repository**

2. **Clone your fork**
   ```bash
   git clone https://github.com/YOUR-USERNAME/sz_kibana_dashboard.git
   cd sz_kibana_dashboard
   ```

3. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

4. **Make your changes**

5. **Test locally**
   ```bash
   # Validate configuration
   docker compose -f kibana.yaml config --quiet

   # Test deployment
   docker compose -f kibana.yaml up -d

   # Run integration tests
   # (see Testing Requirements section)

   # Cleanup
   docker compose -f kibana.yaml down -v
   ```

### Coding Standards

#### YAML Files

- Use 2 spaces for indentation
- Keep lines under 200 characters where possible
- Pass yamllint validation:
  ```bash
  yamllint -c .yamllint kibana.yaml
  ```

#### Documentation (Markdown)

- Use clear, concise language
- Include code examples where appropriate
- Pass markdownlint validation:
  ```bash
  markdownlint *.md
  ```
- Add table of contents for documents > 100 lines
- Use consistent heading levels

#### Docker Compose

- Use explicit service names
- Include comments for complex configurations
- Specify image versions (no `latest` tags in production configs)
- Use environment variables for configurable values
- Include resource limits for production examples

## Pull Request Process

### Before Submitting

1. **Update documentation** if you've changed functionality
2. **Add tests** for new features
3. **Run all tests** and ensure they pass
4. **Update CHANGELOG.md** (if applicable)
5. **Ensure CI passes** (GitHub Actions)

### PR Checklist

- [ ] Code follows project standards
- [ ] All tests pass locally
- [ ] Documentation updated
- [ ] YAML/Markdown linting passes
- [ ] Commit messages are clear and descriptive
- [ ] PR description explains what and why

### PR Template

```markdown
## Description
[Describe what this PR does]

## Motivation and Context
[Why is this change needed? What problem does it solve?]

## Type of Change
- [ ] Bug fix (non-breaking change fixing an issue)
- [ ] New feature (non-breaking change adding functionality)
- [ ] Breaking change (fix or feature causing existing functionality to change)
- [ ] Documentation update

## How Has This Been Tested?
[Describe the tests you ran]

## Screenshots (if appropriate)
[Add screenshots]

## Checklist
- [ ] Code follows project standards
- [ ] Self-reviewed my code
- [ ] Commented complex areas
- [ ] Updated documentation
- [ ] Added tests
- [ ] All tests pass
- [ ] CI/CD pipeline passes
```

### Commit Message Guidelines

Follow conventional commits format:

```
type(scope): subject

body

footer
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

**Examples:**

```
feat(dashboard): add network traffic visualization panel

Add a new panel to the Senzing Dashboard showing network traffic
statistics for better monitoring of GELF log ingestion rates.

Closes #123
```

```
fix(elasticsearch): increase default heap size to 512MB

Previous 256MB default was too low for production workloads,
causing OOM errors. Increased to 512MB for better stability.

Fixes #456
```

```
docs(security): add TLS/SSL configuration examples

Added step-by-step guide for enabling TLS/SSL encryption
on all ELK components.
```

## Testing Requirements

### Manual Testing

Before submitting a PR, test the following:

1. **Clean Deployment**
   ```bash
   docker compose -f kibana.yaml up -d
   # Verify all services start
   docker compose -f kibana.yaml ps
   # Check service health
   curl http://localhost:9200/_cluster/health
   curl http://localhost:5601/api/status
   # Cleanup
   docker compose -f kibana.yaml down -v
   ```

2. **Dashboard Verification**
   ```bash
   # Start stack
   docker compose -f kibana.yaml up -d
   # Wait for dashboard import
   sleep 30
   # Verify dashboard exists
   curl -s "http://localhost:5601/api/saved_objects/dashboard/e37f3b7d-3081-41e4-a9af-01b97c340e14" | grep "Senzing Dashboard"
   ```

3. **Log Ingestion Test**
   ```bash
   # Send test GELF message
   echo '{"version":"1.1","host":"test","short_message":"Test log","level":6,"_container_name":"test"}' | nc -u localhost 12201
   # Wait for indexing
   sleep 10
   # Verify in Elasticsearch
   curl -s "http://localhost:9200/log-*/_search?q=message:Test" | grep -q '"total":{"value":[1-9]'
   ```

### Automated Testing

CI/CD tests run automatically on PR submission (see `.github/workflows/ci.yml`):

- YAML validation
- Docker Compose validation
- Markdown linting
- Full integration test

### Testing Checklist

- [ ] Clean deployment works
- [ ] All services start successfully
- [ ] Dashboard imports correctly
- [ ] Log ingestion works
- [ ] Documentation is accurate
- [ ] No broken links in documentation
- [ ] Configuration examples are valid

## Documentation Standards

### Structure

- Start with clear title and description
- Include table of contents for long documents
- Use appropriate heading hierarchy (H1 → H2 → H3)
- Add "Last Updated" date for time-sensitive content

### Writing Style

- Use active voice
- Be concise but complete
- Include examples for complex concepts
- Use code blocks with syntax highlighting
- Add warnings/notes where appropriate

### Code Examples

Always include:
- Context (what the example does)
- Complete, working code
- Expected output (where relevant)
- Explanation of important parts

```yaml
# Good example:
# This configures Elasticsearch with 1GB heap for medium workloads
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx1g -Xms1g"  # Match Xmx and Xms
```

### Cross-References

Link to related documentation:

```markdown
For security configuration, see [SECURITY.md](SECURITY.md).
```

## Code Review Process

### Review Criteria

Reviewers will check:

1. **Functionality**: Does it work as intended?
2. **Code Quality**: Is it maintainable and well-structured?
3. **Documentation**: Is it properly documented?
4. **Tests**: Are there adequate tests?
5. **Backward Compatibility**: Does it break existing functionality?
6. **Security**: Are there security implications?
7. **Performance**: Does it impact performance?

### Review Timeline

- Initial review within 3-5 business days
- Follow-up on feedback within 1 week
- Merge after approval from at least one maintainer

### Addressing Feedback

- Respond to all review comments
- Make requested changes or explain why not
- Mark conversations as resolved when addressed
- Be open to suggestions and alternative approaches

## Getting Help

If you need help:

1. **Check existing documentation** (README.md, TROUBLESHOOTING.md, etc.)
2. **Search existing issues** for similar questions
3. **Ask in discussions** for general questions
4. **Open an issue** for bugs or specific problems

## Recognition

Contributors will be recognized in:

- CHANGELOG.md (for significant contributions)
- GitHub contributors page
- Project README.md (for major contributions)

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (see LICENSE file).

## Questions?

Feel free to open an issue with questions about contributing. We're here to help!

Thank you for contributing to Senzing ELK Stack! 🎉
