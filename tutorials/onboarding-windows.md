# Environment Setup (Windows)

Read the below instructions carefully and follow them step by step to set up your local working environment.

## Step I: Command-line Terminal Environment

During the course, you'll build and manage apps in a Linux environment (specifically, Ubuntu virtual machines on AWS).

The Linux Terminal will be your best friend. You'll use it to run commands, troubleshoot issues, and interact with servers.

To match the course environment as closely as possible, set up Ubuntu via WSL2 (Windows Subsystem for Linux), which provides a fully functional Ubuntu environment inside Windows.

1. Install WSL.
   Open PowerShell as Administrator and run:

   ```powershell
   wsl --install
   ```

   This installs Ubuntu by default. If not, see the manual install guide:
   https://learn.microsoft.com/en-us/windows/wsl/install

2. Update WSL to Version 2.

   ```powershell
   wsl --set-default-version 2
   ```

3. Launch an Ubuntu terminal by searching for Ubuntu in the Start menu.

While you're using Windows, do everything possible inside Ubuntu on WSL so your setup matches the Linux environment used in the course.

## Step II: Python Interpreter

We'll be using Python as the primary programming language, version 3.10 or higher.

Your system may already have Python installed, but we want to make sure it is the right version and that you also have `pip` and virtual environment tools available.

In your opened WSL Terminal window, run:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```


## Step III: IDE (Integrated Development Environment)

The next and last step is to set up an IDE.
You will use it extensively to write, run, and debug your code.

You are free to use any IDE you like, but we recommend VS Code.

### VS Code

VS Code is lightweight, fast, and works well on Linux, Windows, and macOS.

As a Windows user, install VS Code on the Windows side (not inside WSL).

1. Download and install VS Code:
   https://code.visualstudio.com/Download
2. Install the Remote - WSL extension:
   `vscode:extension/ms-vscode-remote.remote-wsl`
3. Install the Python extension:
   `vscode:extension/ms-python.python`
4. Install the Python Debugger extension:
   `vscode:extension/ms-python.debugpy`
5. Install the Pylint extension:
   `vscode:extension/ms-python.pylint`
6. Set WSL (Ubuntu) as your default terminal profile in VS Code:
   - Open Command Palette (`Ctrl+Shift+P`), run `Terminal: Select Default Profile`, and choose `WSL` or `Ubuntu (WSL)`.
   - Open a new terminal in VS Code and verify it opens in your Ubuntu/WSL shell.


Tip: Turn on Auto Save from File -> Auto Save.

