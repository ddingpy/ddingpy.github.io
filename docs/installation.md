---
layout: default
title: Installation
date: 2024-08-11
description: hello
nav_order: 2
---

# Installation Guide
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Requirements

Before installing, ensure your system meets these requirements:

### System Requirements

- **Operating System**: Windows 10+, macOS 10.14+, or Ubuntu 18.04+
- **Memory**: Minimum 4GB RAM (8GB recommended)
- **Storage**: At least 2GB free disk space
- **Network**: Internet connection for downloading packages

### Software Requirements

- Node.js 14.0 or higher
- npm 6.0 or higher
- Git 2.0 or higher

## Installation Methods

### Method 1: Using npm (Recommended)

The easiest way to install is using npm:

```bash
npm install -g our-tool
```

Verify the installation:

```bash
our-tool --version
```

### Method 2: Using Homebrew (macOS)

For macOS users, you can use Homebrew:

```bash
brew tap our-org/tap
brew install our-tool
```

### Method 3: Manual Installation

1. Download the latest release from [GitHub Releases](https://github.com/yourusername/yourrepository/releases)
2. Extract the archive:
   ```bash
   tar -xzf our-tool-v2.0.0.tar.gz
   ```
3. Move to installation directory:
   ```bash
   sudo mv our-tool /usr/local/bin/
   ```
4. Make it executable:
   ```bash
   chmod +x /usr/local/bin/our-tool
   ```

## Post-Installation Setup

### Configuration

Create a configuration file:

```bash
our-tool init
```

This creates a `.ourtoolrc` file in your home directory.

### Environment Variables

Set the following environment variables:

```bash
export OUR_TOOL_HOME=/path/to/installation
export PATH=$PATH:$OUR_TOOL_HOME/bin
```

Add these to your shell configuration file (`.bashrc`, `.zshrc`, etc.) to make them permanent.

## Verification

Run the following commands to verify your installation:

```bash
# Check version
our-tool --version

# Run diagnostic
our-tool doctor

# Test connection
our-tool ping
```

## Troubleshooting

### Common Issues

#### Permission Denied

If you get permission errors during installation:

```bash
sudo npm install -g our-tool
```

#### Command Not Found

If the command is not found after installation:

1. Check if it's installed:
   ```bash
   which our-tool
   ```
2. Update your PATH:
   ```bash
   export PATH=$PATH:/usr/local/bin
   ```

#### Installation Fails

If installation fails, try:

1. Clear npm cache:
   ```bash
   npm cache clean --force
   ```
2. Use a different registry:
   ```bash
   npm install -g our-tool --registry https://registry.npmjs.org/
   ```

## Updating

To update to the latest version:

```bash
npm update -g our-tool
```

Or if using Homebrew:

```bash
brew upgrade our-tool
```

## Uninstalling

To remove the tool:

```bash
npm uninstall -g our-tool
```

Or with Homebrew:

```bash
brew uninstall our-tool
```

## Next Steps

After successful installation:

1. Read the [Quick Start Guide](quickstart/)
2. Configure your [settings](guide/configuration/)
3. Try the [tutorials](tutorials/)

---

<small>Need help? Contact [support@example.com](mailto:support@example.com)</small>