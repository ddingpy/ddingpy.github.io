---
layout: default
parent: Contents
date: 2025-07-13
nav_exclude: true
---

#  A comprehensive guide to setting up and using Node Version Manager (NVM) on macOS.
- TOC
{:toc}

### Introduction

NVM, or Node Version Manager, is a command-line tool that allows you to install, manage, and switch between multiple versions of Node.js on a single machine. This is particularly useful for developers who work on various projects that may have different Node.js version requirements, ensuring compatibility and preventing version-related conflicts.

### Prerequisites

Before installing NVM, you'll need to have the following:

* **Command Line Tools**: Ensure that Xcode Command Line Tools are installed. You can install them by running the following command in your terminal:
    ```bash
    xcode-select --install
    ```

### Installation Guide

The most straightforward way to install NVM is by using the official install script.

1.  **Run the Install Script**: Open your terminal and execute the following command. This will download and run the NVM installation script.

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    ```

2.  **Configure Your Shell Profile**: After the installation, you need to configure your shell to automatically load NVM every time you open a new terminal window. Depending on your shell, you will need to add the following lines to the corresponding configuration file (`~/.zshrc` for Zsh, which is the default for modern macOS, or `~/.bash_profile` for Bash).

    ```bash
    export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
    ```

3.  **Apply the Changes**: To apply the changes to your current terminal session, you can either close and reopen the terminal or run the `source` command on your configuration file.

    For Zsh:
    ```bash
    source ~/.zshrc
    ```

    For Bash:
    ```bash
    source ~/.bash_profile
    ```

### Verification

To confirm that NVM has been installed correctly, run the following command:

```bash
nvm --version
```

If the installation was successful, you will see the installed NVM version number.

### Basic Usage

Here are some of the most common commands for managing your Node.js versions with NVM.

* **Listing Available Node.js Versions**: To see a list of all the Node.js versions that you can install, use the `ls-remote` command.

    ```bash
    nvm ls-remote
    ```

* **Installing a Specific Node.js Version**: You can install a particular version of Node.js. For example, to install the latest Long-Term Support (LTS) version, which is generally recommended for most projects, use:

    ```bash
    nvm install --lts
    ```

    To install a specific version number, such as `18.17.0`, you would run:

    ```bash
    nvm install 18.17.0
    ```

* **Switching Between Installed Versions**: To switch to a different installed version of Node.js for your current terminal session, use the `use` command.

    ```bash
    nvm use 18.17.0
    ```

* **Setting a Default Node.js Version**: To set a default Node.js version that will be used every time you open a new terminal, use the `alias default` command.

    ```bash
    nvm alias default 18.17.0
    ```