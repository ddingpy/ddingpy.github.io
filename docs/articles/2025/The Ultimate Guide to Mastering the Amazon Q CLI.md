---
layout: default
parent: Contents
date: 2025-07-22
nav_exclude: true
---

# The Ultimate Guide to Mastering the Amazon Q CLI
- TOC
{:toc}

This is a comprehensive guide to using Amazon Q with the Command Line Interface (CLI).

Of course, here is a step-by-step technical guide on how to use Amazon Q with the Command Line Interface (CLI).

## Introduction: What is Amazon Q?

Amazon Q is a generative AI-powered assistant from Amazon Web Services (AWS) designed to help developers and IT professionals work more efficiently. It can answer questions about AWS services, provide code suggestions, help with debugging, and even assist in transforming and upgrading applications. The Amazon Q CLI brings these capabilities directly into your terminal, integrating with your existing command-line workflows.

-----

## Installation

Before you can use the Amazon Q CLI, you need to install it on your local machine. Here's how to do it for different operating systems.

### System Requirements

  * **macOS**: Supported on both Intel and Apple Silicon (ARM64) architectures.
  * **Linux**: Supported on x86\_64 and ARM64 architectures. Requires `glibc` 2.17 or later.
  * **Windows**: Not natively supported, but you can use it with the Windows Subsystem for Linux (WSL).

### Installation Commands

#### macOS (using Homebrew)

The easiest way to install on macOS is with Homebrew.

```bash
brew install amazon-q
```

#### Linux (Manual Installation)

You can download the appropriate binary for your system's architecture and install it manually.

**For x86\_64:**

```bash
curl -o amazon-q-linux-amd64.zip https://releases.amazon-q.aws/q-cli/latest/amazon-q-linux-amd64.zip
unzip amazon-q-linux-amd64.zip
sudo mv amazon-q-linux-amd64 /usr/local/bin/q
```

**For ARM64:**

```bash
curl -o amazon-q-linux-arm64.zip https://releases.amazon-q.aws/q-cli/latest/amazon-q-linux-arm64.zip
unzip amazon-q-linux-arm64.zip
sudo mv amazon-q-linux-arm64 /usr/local/bin/q
```

#### Shell Integration

After installation, it's highly recommended to set up shell integration for features like autocompletion. The CLI will prompt you to do this the first time you run it. Follow the on-screen instructions.

-----

## Authentication

To use Amazon Q, you need to authenticate with your AWS account. You can use either an AWS Builder ID or an IAM Identity Center (formerly AWS SSO) profile.

### Using AWS Builder ID

If you don't have an AWS account or prefer to use a separate identity, you can create a free AWS Builder ID.

1.  Run the following command to start the login process:

    ```bash
    q login
    ```

2.  This will open a browser window asking you to sign in with your AWS Builder ID. If you don't have one, you can create one from this page.

3.  After successful login, you'll be redirected back to your terminal, and the CLI will be authenticated.

### Using IAM Identity Center

If your organization uses IAM Identity Center, you can configure the Amazon Q CLI to use it.

1.  First, ensure you have configured a profile for IAM Identity Center in your AWS config file (`~/.aws/config`). It should look something like this:

    ```ini
    [profile my-sso-profile]
    sso_start_url = https://your-sso-portal.awsapps.com/start
    sso_region = us-east-1
    sso_account_id = 123456789012
    sso_role_name = YourRoleName
    region = us-east-1
    output = json
    ```

2.  Then, you can tell Amazon Q to use this profile:

    ```bash
    q login --profile my-sso-profile
    ```

-----

## Basic Commands

Here are some of the most common commands you'll use with the Amazon Q CLI.

### Starting a Chat

To start an interactive chat session with Amazon Q, simply run:

```bash
q chat
```

You can then ask questions in natural language. For example:

> How do I create an S3 bucket?

### Getting Help with AWS CLI Commands

If you're unsure about an AWS CLI command, you can ask Amazon Q to explain it or even generate it for you.

```bash
q aws "create a new ec2 instance of type t2.micro"
```

This will suggest the appropriate `aws ec2 run-instances` command with the required parameters.

### Getting Help

To see a list of all available commands and options, use the `--help` flag:

```bash
q --help
```

You can also get help for a specific subcommand:

```bash
q chat --help
```

-----

## Use Cases

Amazon Q can be a powerful assistant for various development and operational tasks.

### Debugging AWS CLI Commands

Imagine you have an AWS CLI command that's not working. You can ask Amazon Q for help.

**Example:**

Let's say you're trying to list IAM users but are getting an error. You can ask:

```bash
q aws "I'm trying to list IAM users but I get an AccessDenied error. What could be the issue?"
```

Amazon Q might suggest checking your IAM permissions and provide a link to the relevant documentation.

### Getting Code Snippets

You can ask Amazon Q to generate code snippets in various programming languages.

**Example:**

```bash
q code "write a python function to upload a file to s3"
```

Amazon Q will provide you with a Python function using the Boto3 library to accomplish this.

### Understanding AWS Services

If you're new to an AWS service, Amazon Q can provide a quick overview and links to resources.

**Example:**

```bash
q aws "explain what AWS Lambda is and when I should use it"
```

-----

## Tips for Power Users

  * **Command Chaining:** You can pipe the output of other commands to Amazon Q for analysis. For example, to analyze a CloudFormation template for potential issues:
    ```bash
    cat my-template.yaml | q aws "review this CloudFormation template for security vulnerabilities"
    ```
  * **Aliases:** Create shell aliases for frequently used commands to save time. For example, in your `.bashrc` or `.zshrc`:
    ```bash
    alias qa="q aws"
    ```
  * **Contextual Awareness:** The Amazon Q CLI is aware of your current directory. If you're in a Git repository, it can use that context to provide more relevant answers.

-----

## Troubleshooting

Here are some common issues you might encounter and how to resolve them.

### `q: command not found`

This error means the `q` executable is not in your system's `PATH`.

  * **Solution:** Ensure that the directory where you installed the `q` binary (e.g., `/usr/local/bin`) is in your `PATH` environment variable. You can check your `PATH` with `echo $PATH`. If it's not there, add it to your shell's startup file (e.g., `~/.bashrc`, `~/.zshrc`).

### Authentication Errors

If you're having trouble logging in:

  * **Check Credentials:** Double-check that your AWS Builder ID credentials or IAM Identity Center profile are correct.
  * **Firewall/Proxy:** If you are behind a corporate firewall or proxy, you may need to configure your environment variables (`HTTPS_PROXY`, `HTTP_PROXY`) for the CLI to connect to AWS services.
  * **Token Expiration:** Authentication tokens expire. If you get an authentication error after a period of inactivity, simply run `q login` again to refresh your credentials.

### `Error: Insufficient permissions`

This error indicates that the IAM role or user you are authenticated as does not have the necessary permissions to perform the requested action.

  * **Solution:** Review the IAM policy attached to your user or role and ensure it grants the required permissions for the AWS service you are trying to interact with. You can ask Amazon Q to help you understand the necessary permissions. For instance: `q aws "what IAM permissions are needed to create an S3 bucket?"`