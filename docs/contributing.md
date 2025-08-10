---
layout: default
title: Contributing
nav_order: 6
nav_exclude: true
---

# Contributing Guide
{: .no_toc }

Help us improve our documentation and platform
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Welcome Contributors!

We're thrilled that you're interested in contributing! This guide will help you get started with contributing to our documentation and codebase.

## Ways to Contribute

### 📝 Documentation

Help improve our documentation:
- Fix typos and grammar
- Clarify confusing sections
- Add missing information
- Create new tutorials
- Translate documentation

### 🐛 Bug Reports

Found a bug? Let us know:
- Search existing issues first
- Create detailed bug reports
- Include reproduction steps
- Provide system information

### 💡 Feature Requests

Have an idea? We'd love to hear it:
- Describe the problem it solves
- Explain your proposed solution
- Discuss alternatives considered

### 💻 Code Contributions

Contribute code:
- Fix bugs
- Add new features
- Improve performance
- Write tests
- Refactor code

## Getting Started

### Step 1: Fork the Repository

1. Visit [our GitHub repository](https://github.com/yourusername/yourrepository)
2. Click the "Fork" button
3. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/yourrepository.git
   cd yourrepository
   ```

### Step 2: Set Up Development Environment

Install dependencies:

```bash
# Install project dependencies
npm install

# Install documentation dependencies
bundle install
```

Set up pre-commit hooks:

```bash
npm run setup-hooks
```

### Step 3: Create a Branch

Create a feature branch:

```bash
git checkout -b feature/your-feature-name
```

Branch naming conventions:
- `feature/` - New features
- `fix/` - Bug fixes
- `docs/` - Documentation updates
- `refactor/` - Code refactoring
- `test/` - Test improvements

## Making Changes

### Documentation Changes

1. Edit Markdown files in the `docs/` directory
2. Preview changes locally:
   ```bash
   bundle exec jekyll serve
   ```
3. Check for broken links:
   ```bash
   npm run check-links
   ```

### Code Changes

1. Write your code following our style guide
2. Add/update tests
3. Run tests:
   ```bash
   npm test
   ```
4. Check code style:
   ```bash
   npm run lint
   ```

## Code Style Guide

### JavaScript/TypeScript

We use ESLint with Prettier:

```javascript
// Good
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Bad
function calculateTotal(items){
  return items.reduce((sum,item)=>sum+item.price,0)
}
```

Key points:
- Use 2 spaces for indentation
- Use semicolons
- Use single quotes for strings
- Add trailing commas
- Maximum line length: 100 characters

### Markdown

For documentation:
- Use ATX-style headers (`#`)
- Wrap lines at 80 characters (except code blocks)
- Use fenced code blocks with language hints
- Add blank lines around headers and lists

Example:

```markdown
# Header

This is a paragraph with some **bold** and *italic* text.

## Subheader

- List item 1
- List item 2

\```javascript
// Code block
const example = "code";
\```
```

## Commit Guidelines

We follow [Conventional Commits](https://www.conventionalcommits.org/):

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Test changes
- `chore`: Build process or auxiliary tool changes

### Examples

```bash
# Feature
git commit -m "feat(api): add user authentication endpoint"

# Bug fix
git commit -m "fix(parser): handle empty input correctly"

# Documentation
git commit -m "docs(readme): update installation instructions"
```

## Submitting Changes

### Pull Request Process

1. **Push your branch**:
   ```bash
   git push origin feature/your-feature-name
   ```

2. **Create Pull Request**:
   - Go to the original repository
   - Click "New Pull Request"
   - Select your branch
   - Fill out the PR template

3. **PR Description Template**:
   ```markdown
   ## Description
   Brief description of changes

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Documentation update
   - [ ] Performance improvement

   ## Testing
   - [ ] Tests pass locally
   - [ ] Added new tests
   - [ ] Updated documentation

   ## Screenshots (if applicable)
   Add screenshots here

   ## Related Issues
   Fixes #123
   ```

### Review Process

1. **Automated Checks**: CI/CD runs tests and linting
2. **Code Review**: Maintainers review code
3. **Feedback**: Address any requested changes
4. **Approval**: Two approvals required
5. **Merge**: Maintainer merges PR

## Testing

### Running Tests

```bash
# Run all tests
npm test

# Run specific test file
npm test -- path/to/test.js

# Run with coverage
npm run test:coverage

# Run in watch mode
npm run test:watch
```

### Writing Tests

Example test:

```javascript
describe('Calculator', () => {
  describe('add()', () => {
    it('should add two numbers correctly', () => {
      expect(add(2, 3)).toBe(5);
    });

    it('should handle negative numbers', () => {
      expect(add(-1, 1)).toBe(0);
    });
  });
});
```

## Documentation

### Building Documentation

```bash
# Serve locally
bundle exec jekyll serve

# Build for production
bundle exec jekyll build

# Check for broken links
npm run check-docs
```

### Writing Documentation

Guidelines:
- Write for beginners
- Include examples
- Use clear, simple language
- Add screenshots when helpful
- Test all code examples

## Community

### Code of Conduct

We follow our [Code of Conduct](CODE_OF_CONDUCT.md). Be respectful and inclusive.

### Getting Help

- 💬 [Discord](https://discord.gg/example) - Chat with contributors
- 📧 [Email](mailto:contributors@example.com) - Direct questions
- 🗣️ [Discussions](https://github.com/yourusername/yourrepository/discussions) - General discussions

### Recognition

Contributors are recognized in:
- [CONTRIBUTORS.md](CONTRIBUTORS.md)
- Release notes
- Annual contributor spotlight

## Release Process

Our release process:

1. **Version Bump**: Update version numbers
2. **Changelog**: Update CHANGELOG.md
3. **Testing**: Full test suite passes
4. **Documentation**: Update relevant docs
5. **Tag**: Create git tag
6. **Release**: Publish to npm/PyPI
7. **Announce**: Blog post and social media

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (MIT License).

## Resources

### Helpful Links

- [Issue Tracker](https://github.com/yourusername/yourrepository/issues)
- [Project Board](https://github.com/yourusername/yourrepository/projects)
- [Wiki](https://github.com/yourusername/yourrepository/wiki)
- [Roadmap](https://github.com/yourusername/yourrepository/roadmap)

### Learning Resources

- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [How to Contribute to Open Source](https://opensource.guide/how-to-contribute/)
- [Markdown Guide](https://www.markdownguide.org/)
- [JavaScript Style Guide](https://github.com/airbnb/javascript)

## Thank You!

Thank you for contributing! Your efforts help make our project better for everyone. 🎉

---

<small>Questions? Contact [contributors@example.com](mailto:contributors@example.com)</small>