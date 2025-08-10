---
layout: default
parent: Contents
date: 2025-07-12
nav_exclude: true
---
# Sublime Text Command line Interface
- TOC
{:toc}

https://www.sublimetext.com/docs/command_line.html

## ZSH

If using Zsh, the default starting with macOS 10.15, the following command will add the bin folder to the PATH environment variable:

```
echo 'export PATH="/Applications/Sublime Text.app/Contents/SharedSupport/bin:$PATH"' >> ~/.zprofile
```