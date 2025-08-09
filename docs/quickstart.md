---
layout: single
title: "Quick Start Guide"
permalink: /quickstart/
classes: wide
#toc: true
#toc_label: "Quick Start"
#toc_icon: "rocket"
#toc_sticky: true
header:
  teaser: https://via.placeholder.com/600x400
---

Get up and running in just 5 minutes with our step-by-step quick start guide.

## Overview

This quick start guide will help you get started with our platform in just a few minutes. By the end of this guide, you'll have:

- ✅ Installed the necessary tools
- ✅ Created your first project
- ✅ Run your first command
- ✅ Understood the basic concepts

## Step 1: Install

First, install our tool using npm:

```bash
npm install -g our-tool
```

**Note:** Requires Node.js 14.0 or higher. See [Installation Guide](/installation/) for other methods.
{: .notice--info}

## Step 2: Initialize

Create a new project:

```bash
mkdir my-project
cd my-project
our-tool init
```

This creates the following structure:

```
my-project/
├── config.yml
├── src/
│   └── main.js
├── tests/
│   └── test.js
└── README.md
```

## Step 3: Configure

Edit `config.yml` to set your preferences:

```yaml
name: my-project
version: 1.0.0
settings:
  debug: false
  output: ./dist
  format: json
```

Key configuration options:

| Option | Description | Default |
|:-------|:------------|:--------|
| `name` | Project name | Required |
| `version` | Project version | 1.0.0 |
| `settings.debug` | Enable debug mode | false |
| `settings.output` | Output directory | ./dist |
| `settings.format` | Output format | json |

## Step 4: Write Your First Script

Create a simple script in `src/hello.js`:

```javascript
// src/hello.js
module.exports = function hello(name) {
  return `Hello, ${name}!`;
};
```

## Step 5: Run

Execute your script:

```bash
our-tool run src/hello.js --name "World"
```

Expected output:

```
Hello, World!
```

## Basic Commands

Here are the essential commands you'll use:

### Initialize a Project

```bash
our-tool init [options]
```

Options:
- `--template <name>` - Use a specific template
- `--force` - Overwrite existing files

### Run Scripts

```bash
our-tool run <file> [options]
```

Options:
- `--watch` - Watch for file changes
- `--debug` - Enable debug output

### Build Project

```bash
our-tool build [options]
```

Options:
- `--production` - Optimize for production
- `--minify` - Minify output

### Test

```bash
our-tool test [pattern]
```

Run tests matching the pattern.

## Working with Data

### Input Data

Our tool accepts various input formats:

#### JSON Input

```bash
echo '{"name": "Alice"}' | our-tool run script.js
```

#### File Input

```bash
our-tool run script.js --input data.json
```

#### Environment Variables

```bash
MY_VAR=value our-tool run script.js
```

### Output Formats

Specify output format with `--format`:

```bash
# JSON output (default)
our-tool run script.js --format json

# Plain text
our-tool run script.js --format text

# CSV
our-tool run script.js --format csv
```

## Examples

### Example 1: Data Processing

Process a CSV file:

```javascript
// process.js
const data = require('./data.csv');

module.exports = function process() {
  return data.map(row => ({
    ...row,
    processed: true,
    timestamp: new Date()
  }));
};
```

Run:

```bash
our-tool run process.js --output results.json
```

### Example 2: API Integration

Make API calls:

```javascript
// api.js
const fetch = require('node-fetch');

module.exports = async function fetchData() {
  const response = await fetch('https://api.example.com/data');
  return response.json();
};
```

Run:

```bash
our-tool run api.js --async
```

### Example 3: Batch Processing

Process multiple files:

```bash
for file in *.txt; do
  our-tool run process.js --input "$file" --output "processed_$file"
done
```

## Best Practices

1. **Use Version Control**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   ```

2. **Environment Configuration**
   - Use `.env` files for sensitive data
   - Never commit secrets to repository

3. **Testing**
   - Write tests for your scripts
   - Run tests before deployment

4. **Documentation**
   - Document your configuration
   - Add comments to complex scripts

## Common Patterns

### Pattern 1: Pipeline

Chain multiple operations:

```bash
our-tool run extract.js | \
our-tool run transform.js | \
our-tool run load.js
```

### Pattern 2: Parallel Processing

Process files in parallel:

```bash
our-tool run process.js --parallel --workers 4
```

### Pattern 3: Scheduled Tasks

Run periodic tasks:

```bash
# Using cron
0 * * * * our-tool run hourly-task.js
```

## Troubleshooting

### Debug Mode

Enable verbose output:

```bash
our-tool run script.js --debug
```

### Check Configuration

Validate your configuration:

```bash
our-tool config validate
```

### View Logs

Check execution logs:

```bash
our-tool logs --tail 50
```

## Next Steps

Now that you've completed the quick start:

1. 📖 Read the [User Guide](../guide/) for detailed documentation
2. 🔧 Explore [Configuration Options](../guide/configuration/)
3. 📚 Check out [Tutorials](../tutorials/) for advanced examples
4. 🚀 Learn [Best Practices](../guide/best-practices/)

## Getting Help

- 📧 Email: [support@example.com](mailto:support@example.com)
- 💬 Chat: Available in dashboard
- 📚 Documentation: You're here!
- 🐛 Issues: [GitHub](https://github.com/yourusername/yourrepository/issues)

---

<small>Congratulations! You've completed the quick start guide. 🎉</small>