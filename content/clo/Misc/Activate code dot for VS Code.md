+++
title = "Activate `code .` for VS Code"
weight = 1
date = 2024-02-10
draft = true
+++

To make the command `code .` work on a Mac to start Visual Studio Code (VS Code), you need to install the command line tools that come with VS Code. This allows you to launch VS Code from the terminal. Here's how you can set it up:

1. **Open VS Code**:
   First, make sure that Visual Studio Code is installed on your Mac. Open it from your Applications folder or Launchpad.

2. **Open the Command Palette**:
   With VS Code open, access the Command Palette by using the shortcut `Cmd + Shift + P` (⌘ + Shift + P).

3. **Install `code` Command in PATH**:
   In the Command Palette, start typing `Shell Command: Install 'code' command in PATH` and select it when it appears. After selecting this command, VS Code will automatically install the `code` command in your system's PATH.

4. **Verify Installation**:
   Close the terminal if it's open, and then reopen it to refresh the environment variables. You can verify that the `code` command is correctly installed by typing `code --version` in the terminal. This should display the version of VS Code that you have installed.

5. **Usage**:
   Now, you can use the `code` command in the terminal to open VS Code. For example:
   - `code .` will open VS Code in the current directory.
   - `code /path/to/folder` will open VS Code in the specified folder.
   - `code /path/to/file` will open the specified file in VS Code.

If you encounter any issues with these steps, ensure that your VS Code application is up to date, and try repeating the process. Additionally, you might want to manually add the path to VS Code's bin folder to your `.bash_profile`, `.zshrc`, or equivalent shell configuration file if for some reason the automatic process doesn't work. This is less common but can be necessary in some configurations:

```bash
export PATH="$PATH:/Applications/Visual Studio Code.app/Contents/Resources/app/bin"
```

Add this line to your shell's profile file (e.g., `~/.bash_profile`, `~/.zshrc`, etc.), then source the file (`source ~/.bash_profile` or `source ~/.zshrc`) or restart your terminal to apply the changes.