# Python Development Environment

## Get the code 

The first thing you'll probably do in your first day at work is to set up your development environment and get the code base of the project you'll be working on.

Throughout the course, we'll be working on a project called PolyAI, which is an AI agent for image processing. 

Let's create your own copy of the PolyAI project.

1. Navigate to the template repository: https://github.com/alonitac/PolyAIFursa
2. Click **Use this template** → **Create a new repository**
3. Name your repository (e.g., `PolyAI`)
4. Click **Create repository**

Now clone your repository to your local machine:

```bash
git clone https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
cd YOUR-REPO-NAME
```

Now open the project in VSCode by **File** → **Open Folder** → Select your project folder.

## Understanding Python Execution Environments

A **Python execution environment** is where your Python code runs. It includes:

- The Python interpreter (the program that executes Python code)
- Installed packages and libraries
- Environment variables and configurations

When you run `which python`, you're checking which Python interpreter is active in your current environment.

You can open a terminal in VSCode by **Terminal** → **New Terminal**, and check the active Python interpreter.

### Create a Virtual Environment (venv)

In each Python project, we would like to have a separate environment to manage dependencies. This is called a **virtual environment (venv)**.
A **virtual environment** is an isolated Python environment for your project. It prevents dependency conflicts between different projects.

In the VSCode terminal, create a virtual environment by executing the following command from the **root directory of your project**:

```bash
python -m venv .venv
```

The `python -m venv` command executes the built-in `venv` module to create a new virtual environment.
The `.venv` argument specifies the **directory name** where the virtual environment will be created.

You'll notice a new folder named `.venv` in your project directory. This folder contains the isolated Python interpreter and all installed packages for this project.

To work with Python virtual environments, you need to **activate** them.

VScode may automatically detect the venv and suggest associating it for the workspace. Accept it. 
If not, you can manually select it by pressing `Ctrl+Shift+P` (or `Cmd+Shift+P` on macOS), typing **Python: Select Interpreter**, and choosing the one from your `.venv` folder.

To make sure the virtual environment is activated, please open a new terminal in VSCode, you should see `(.venv)` in your terminal prompt, indicating the virtual environment is active.

### Install Dependencies with `pip`

**pip** is Python's package installer.
It downloads and installs libraries from the Python Package Index (PyPI).

The PolyAI project has multiple microservices. We'll talk about microservices later on in the course, but for now let's follow the instructions [in the README](https://github.com/alonitac/PolyAIFursa/blob/main/services/yolo/README.md) of the `Yolo` microservice to install the required dependencies and run the service.

#### Debugging in VSCode

A much better way to run and debug Python code is using the built-in debugger in VSCode.

Debugging lets you pause execution, inspect variables, and step through code line-by-line.

1. Open `app.py` file in VSCode.
2. Click the **Run and Debug** icon in the left sidebar (or press `Ctrl+Shift+D`)
3. Click **Run and Debug** button.
4. In the opened panel, choose **Python File**.
5. The debugger will start, and you can set breakpoints by clicking in the gutter next to the line numbers.
