---
layout: default
title: Getting Started
parent: User Guide
nav_order: 1
---

# Getting Started
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## System Requirements

### Minimum Requirements

- **Operating System**: Windows 10+, macOS 10.14+, Ubuntu 18.04+
- **Memory**: 4GB RAM
- **Storage**: 2GB available space
- **Network**: Broadband internet connection

### Recommended Requirements

- **Operating System**: Latest stable version
- **Memory**: 8GB RAM or more
- **Storage**: 10GB available space
- **Network**: High-speed internet for optimal performance

## Installation

### Windows Installation

1. Download the installer from [our website](https://example.com/download)
2. Run the `.exe` file
3. Follow the installation wizard
4. Add to PATH (optional but recommended)

```powershell
# Verify installation
example --version
```

### macOS Installation

#### Using Homebrew (Recommended)

```bash
brew tap example/tap
brew install example
```

#### Manual Installation

```bash
curl -fsSL https://example.com/install.sh | sh
```

### Linux Installation

#### Ubuntu/Debian

```bash
sudo apt update
sudo apt install example
```

#### Fedora/RHEL

```bash
sudo dnf install example
```

#### From Source

```bash
git clone https://github.com/example/example.git
cd example
make install
```

## Initial Setup

### Step 1: Create an Account

Visit [https://example.com/signup](https://example.com/signup) and create your account.

### Step 2: Generate API Keys

1. Log in to your dashboard
2. Navigate to **Settings** → **API Keys**
3. Click **Generate New Key**
4. Copy and save your key securely

{: .warning }
> **Security Note:** Treat your API keys like passwords. Never share them or commit them to version control.

### Step 3: Configure CLI

Run the configuration wizard:

```bash
example config init
```

This will prompt you for:
- API key
- Default region
- Output format preference

### Step 4: Verify Setup

Test your configuration:

```bash
example auth test
```

Expected output:
```
✓ Authentication successful
✓ API connection established
✓ Configuration valid
```

## Your First Project

### Creating a Project

```bash
example create project my-project
cd my-project
```

### Project Structure

```
my-project/
├── .example/          # Configuration directory
│   └── config.yml     # Project configuration
├── src/               # Source code
│   └── main.js        # Entry point
├── tests/             # Test files
├── docs/              # Documentation
├── package.json       # Dependencies
└── README.md          # Project readme
```

### Running Locally

Start the development server:

```bash
example dev
```

Access your application at `http://localhost:3000`

### Making Changes

1. Edit files in the `src/` directory
2. Changes auto-reload in development mode
3. Check the console for errors

## Basic Commands

### Essential Commands

| Command | Description |
|:--------|:------------|
| `example help` | Show all commands |
| `example create` | Create new resources |
| `example list` | List existing resources |
| `example delete` | Remove resources |
| `example deploy` | Deploy to production |
| `example logs` | View application logs |
| `example status` | Check service status |

### Command Examples

```bash
# Create a new function
example create function hello-world

# List all projects
example list projects

# Deploy to staging
example deploy --env staging

# View real-time logs
example logs --follow
```

## Environment Management

### Development Environment

```bash
example env set development
```

Features:
- Hot reload enabled
- Verbose logging
- Debug mode active

### Staging Environment

```bash
example env set staging
```

Features:
- Production-like setup
- Performance monitoring
- Integration testing enabled

### Production Environment

```bash
example env set production
```

Features:
- Optimized performance
- Security hardening
- Monitoring and alerts

## Next Steps

Now that you have everything set up:

1. **Explore Features**: Check out the Features Guide
2. **Learn Best Practices**: Read our Best Practices guide
3. **Try Tutorials**: Complete the [Tutorial Series](/docs/tutorials/)
4. **Join Community**: Connect with other users in our [Forum](https://forum.example.com)

## Troubleshooting

### Common Issues

**Installation fails on Windows**
- Run as Administrator
- Disable antivirus temporarily
- Check Windows Defender settings

**Permission denied on macOS/Linux**
```bash
sudo chown -R $(whoami) /usr/local/share/example
```

**API key not recognized**
- Check for extra spaces
- Verify key is active in dashboard
- Regenerate if necessary

### Getting Help

If you encounter issues:

1. Check the Troubleshooting Guide
2. Search the [FAQ](/docs/faq/)
3. Ask in the [Community Forum](https://forum.example.com)
4. Contact [Support](mailto:support@example.com)

---

<div class="code-example" markdown="1">
**Tip:** Enable debug mode for detailed error messages: `export DEBUG=true`
</div>