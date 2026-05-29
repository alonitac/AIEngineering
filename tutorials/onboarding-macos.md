# Environment Setup (macOS)

Read the below instructions carefully and follow them step by step to set up your local working environment.

## Step I: Command-line Terminal Environment

During the course, you'll build and manage apps in a Linux environment (specifically, Ubuntu virtual machines on AWS).

The Linux Terminal will be your best friend. You'll use it to run commands, troubleshoot issues, and interact with servers.

macOS (Intel or Apple Silicon) is close enough to Linux for most tools to work well, with a few differences:

- Use `brew` (Homebrew) instead of `apt` to install packages.
- Some tools may require extra configuration due to system differences.

To open a Terminal session, press Cmd + Space to open Spotlight Search, type Terminal, and press Enter.

## Step II: Python Interpreter

We'll be using Python as the primary programming language, version 3.10 or higher.

Your system may already have Python installed, but we want to make sure it is the right version and that you also have `pip` and virtual environment tools available.

macOS does not come with an up-to-date version of Python 3 by default.

Install Python 3 using Homebrew:

```bash
brew install python
```


## Step III: IDE (Integrated Development Environment)

The next and last step is to set up an IDE.
You will use it extensively to write, run, and debug your code.

You are free to use any IDE you like, but we recommend VS Code.

### VS Code

VS Code is lightweight, fast, and works well on Linux, Windows, and macOS.

1. Download and install VS Code:
   https://code.visualstudio.com/Download
2. Navigate to your Yolo project directory.
3. Launch VS Code in that directory:

   ```bash
   code .
   ```

This opens the YoloService project in VS Code.

Tip: Turn on Auto Save from File -> Auto Save.

