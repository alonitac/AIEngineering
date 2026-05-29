# Environment Setup (Linux)

Read the below instructions carefully and follow them step by step to set up your local working environment.

## Step I: Command-line Terminal Environment

During the course, you'll build and manage apps in a Linux environment (specifically, Ubuntu virtual machines on AWS).

The Linux Terminal will be your best friend. You'll use it to run commands, troubleshoot issues, and interact with servers.

Ubuntu is the recommended distribution, but other distros will work too as long as you're comfortable using the terminal.

To open a terminal session, press Ctrl + Alt + T or search for Terminal in your applications.

## Step II: Python Interpreter

We'll be using Python as the primary programming language, version 3.10 or higher.

Your system may already have Python installed, but we want to make sure it is the right version and that you also have `pip` and virtual environment tools available.

In your opened Terminal window, run:

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

Download and install VS Code:
   https://code.visualstudio.com/Download

Tip: Turn on Auto Save from File -> Auto Save.

