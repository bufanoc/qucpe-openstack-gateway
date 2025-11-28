# Contributing to QuCPE-OpenStack Gateway

Thank you for your interest in contributing! This project aims to document and enable the use of QNAP QuCPE devices as OpenStack L2 gateways and edge compute nodes.

## Ways to Contribute

### Documentation
- Improve existing documentation
- Add troubleshooting tips
- Translate documentation
- Add diagrams and visual aids

### Testing
- Test with different OpenStack versions (Wallaby, Xena, Yoga, Zed, 2023.x, 2024.x)
- Test with different QuCPE models
- Validate procedures on fresh deployments
- Performance benchmarking

### Code
- Scripts for automation
- Ansible playbooks for deployment
- Monitoring configurations
- CI/CD pipelines

### Bug Reports
- Document issues encountered
- Provide reproduction steps
- Include logs and configuration

## Getting Started

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Test your changes
5. Commit (`git commit -m 'Add amazing feature'`)
6. Push (`git push origin feature/amazing-feature`)
7. Open a Pull Request

## Pull Request Guidelines

- Use clear, descriptive commit messages
- Update documentation for any changed functionality
- Include test results where applicable
- Reference any related issues

## Code of Conduct

- Be respectful and inclusive
- Provide constructive feedback
- Help others learn

## Questions?

Open an issue with the `question` label or start a discussion.

## Hardware Access

If you have access to QuCPE hardware and want to help test:
- Document your hardware model and QNE version
- Share test results (sanitize any sensitive data)
- Report compatibility issues

## OpenStack Versions Tested

Help us expand this list:

| OpenStack Release | Status | Tester | Notes |
|-------------------|--------|--------|-------|
| Wallaby | Untested | | |
| Xena | Untested | | |
| Yoga | Untested | | |
| Zed | Untested | | |
| 2023.1 (Antelope) | Untested | | |
| 2023.2 (Bobcat) | Untested | | |
| 2024.1 (Caracal) | Untested | | |

## QuCPE Models Tested

| Model | QNE Version | Status | Notes |
|-------|-------------|--------|-------|
| QuCPE-7012 | 1.0.6.q322 | In Progress | Primary development device |
| QuCPE-3032 | | Untested | |
| QuCPE-3034 | | Untested | |

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
