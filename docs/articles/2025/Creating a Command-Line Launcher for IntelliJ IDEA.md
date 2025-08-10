---
layout: default
parent: Contents
date: 2025-07-11
nav_exclude: true
---
# Creating a Command-Line Launcher for IntelliJ IDEA
- TOC
{:toc}

You can run IntelliJ IDEA from the command line to open projects, files, and perform various other actions directly from your terminal. This can be a more efficient workflow for many developers. Here's a guide on how to do it on different operating systems.

### **Creating a Command-Line Launcher**

For ease of use, it's highly recommended to create a command-line launcher. This allows you to simply type `idea` in your terminal to run IntelliJ IDEA.

* **During Installation:** When you first install IntelliJ IDEA, you may be presented with an option to create a command-line launcher. Selecting this is the easiest method.
* **Using the Toolbox App:** If you use the JetBrains Toolbox App to manage your IDEs, you can enable the "Shell scripts" feature. Go to the Toolbox App settings, and under "Tools," ensure that "Shell scripts" is enabled and a path for the scripts is set. This will automatically create the necessary launcher scripts.
* **Manual Creation:**
    * **Windows:** Add the `bin` folder of your IntelliJ IDEA installation directory (e.g., `C:\Program Files\JetBrains\IntelliJ IDEA\bin`) to your system's `PATH` environment variable. This will allow you to run `idea64.exe` from any command prompt.
    * **macOS:** Create a shell script. Open your terminal and create a file named `idea` in a directory that is in your `PATH` (e.g., `/usr/local/bin/`). Add the following content to the file:
        ```sh
        #!/bin/sh
        open -a "IntelliJ IDEA.app" --args "$@"
        ```
        Make the script executable by running `chmod +x /usr/local/bin/idea`.
    * **Linux:** Create a symbolic link to the `idea.sh` script in your IntelliJ IDEA installation's `bin` directory. For example:
        ```sh
        sudo ln -s /path/to/your/idea/installation/bin/idea.sh /usr/local/bin/idea
        ```

---

### **Basic Command-Line Usage**

Once you have the command-line launcher set up, you can use the following commands:

* **Open a Project:** To open a project, simply pass the path to the project's directory as an argument:
    ```bash
    idea /path/to/your/project
    ```

* **Open a Specific File:** You can also open a specific file. IntelliJ IDEA will open the file in the context of its project.
    ```bash
    idea /path/to/your/project/src/main/java/com/example/MyClass.java
    ```

* **Open a File at a Specific Line and Column:** To open a file and place the cursor at a specific line and column, use the `--line` and `--column` arguments:
    ```bash
    idea --line 42 --column 10 /path/to/your/file.txt
    ```

---

### **Advanced Command-Line Features**

IntelliJ IDEA offers several other command-line functionalities:

* **Diffing Files:** You can use IntelliJ IDEA's diff tool to compare two files:
    ```bash
    idea diff /path/to/file1 /path/to/file2
    ```

* **Merging Files:** IntelliJ IDEA can also be used as a three-way merge tool:
    ```bash
    idea merge /path/to/local /path/to/remote /path/to/base /path/to/merged
    ```

For a complete list of command-line arguments and options, you can refer to the official JetBrains documentation.