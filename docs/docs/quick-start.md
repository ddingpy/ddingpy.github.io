---
layout: default
title: Quick Start
nav_order: 2
---

# Quick Start Guide
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

This guide will help you get started with our platform in just 5 minutes. By the end of this tutorial, you'll have:

- ✅ Installed the necessary tools
- ✅ Created your first project
- ✅ Made your first API call
- ✅ Deployed a simple application

## Prerequisites

Before you begin, ensure you have:

- **Node.js** version 14.0 or higher
- **npm** or **yarn** package manager
- A text editor (we recommend VS Code)
- Basic knowledge of JavaScript

## Step 1: Installation

Open your terminal and run:

```bash
npm install -g @example/cli
```

Verify the installation:

```bash
example --version
# Output: example-cli version 2.0.0
```

## Step 2: Create Your First Project

Create a new project using our CLI:

```bash
example create my-first-app
cd my-first-app
```

This creates a basic project structure:

```
my-first-app/
├── src/
│   ├── index.js
│   └── config.js
├── tests/
│   └── index.test.js
├── package.json
└── README.md
```

## Step 3: Configure Your Environment

Create a `.env` file in your project root:

```bash
touch .env
```

Add your configuration:

```env
API_KEY=your_api_key_here
API_URL=https://api.example.com
NODE_ENV=development
```

{: .warning }
> **Important:** Never commit your `.env` file to version control. Add it to `.gitignore`.

## Step 4: Write Your First Code

Open `src/index.js` and add:

```javascript
const { Client } = require('@example/sdk');

// Initialize the client
const client = new Client({
  apiKey: process.env.API_KEY,
  baseUrl: process.env.API_URL
});

// Make your first API call
async function main() {
  try {
    const response = await client.hello();
    console.log('Success!', response);
  } catch (error) {
    console.error('Error:', error.message);
  }
}

main();
```

## Step 5: Run Your Application

Start your application:

```bash
npm start
```

You should see:

```
Success! { message: 'Hello, World!', timestamp: 1234567890 }
```

## Step 6: Deploy Your Application

Deploy to our cloud platform:

```bash
example deploy --production
```

Your application is now live at: `https://my-first-app.example.com`

## What's Next?

Congratulations! You've successfully:
- Created your first project
- Made an API call
- Deployed to production

### Recommended Next Steps

1. **[User Guide](/docs/user-guide/)** - Deep dive into all features
2. **[API Reference](/docs/api/)** - Explore all available endpoints
3. **Best Practices** - Learn optimal patterns
4. **[Tutorials](/docs/tutorials/)** - Build real-world applications

## Common Issues

### Installation Fails

If you encounter permission errors:

```bash
sudo npm install -g @example/cli
```

Or use a Node version manager like `nvm`.

### API Key Invalid

Ensure your API key is:
- Correctly copied (no extra spaces)
- Active in your dashboard
- Has the necessary permissions

### Port Already in Use

Change the default port:

```bash
PORT=3001 npm start
```

## Get Help

Need assistance? We're here to help:

- 📚 [Documentation](/)
- 💬 [Community Forum](https://forum.example.com)
- 📧 [Support Email](mailto:support@example.com)
- 🐛 [Report Issues](https://github.com/example/repo/issues)

---

<div class="code-example" markdown="1">
**Pro Tip:** Join our [Discord server](https://discord.gg/example) for real-time help from the community!
</div>