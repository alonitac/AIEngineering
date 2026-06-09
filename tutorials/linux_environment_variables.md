# Environment variables

## Environment variables defined

Global variables or **environment variables** are variables available for any process or application running in the same environment. Global variables are being transferred from parent process to child program. They are used to store system-wide settings and configuration information, such as the current user's preferences, system paths, and language settings. Environment variables are an essential part of the Unix and Linux operating systems and are used extensively by command-line utilities and scripts.

The `env` or `printenv` commands can be used to display environment variables.

## The `$PATH` environment variable

When you want the system to execute a command, you almost never have to give the full path to that command. For example, we know that the `ls` command is actually an executable file, located in the `/bin` directory (check with `which ls`), yet we don't have to enter the command `/bin/ls` for the computer to list the content of the current directory.

The `$PATH` environment variable is a list of directories separated by colons (`:`) that the shell searches when you enter a command. When you enter a command in the shell, the shell looks for an executable file with that name in each directory listed in the `$PATH` variable, in order. If it finds an executable file with that name, it runs it.

System commands are normal programs that exist in compiled form (e.g. `ls`, `mkdir` etc... ).

```console
myuser@hostname:~$ which ls
/bin/ls
myuser@hostname:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:.....
```

The above example shows that `ls` is actually an executable file located under `/bin/ls`. The `/bin` path is part of the PATH env var, thus we are able to type `ls` shortly.

## The `export` command

The `export` command sets a variable in the current shell **and passes it to any child process** spawned from it. This is how processes receive configuration from the environment.

Recall that the YOLO app reads the confidence threshold from an environment variable:

```python
_raw_threshold = os.environ.get("CONFIDENCE_THRESHOLD")
```

When you run `python app.py`, a new child process is created. For that process to see `CONFIDENCE_THRESHOLD`, you must export it:

```bash
export CONFIDENCE_THRESHOLD=0.7
python app.py
```

If you set it without `export`, the variable exists only in the current shell and the child process (`python app.py`) will not inherit it:

```bash
CONFIDENCE_THRESHOLD=0.7   # NOT visible to child processes
python app.py              # app.py falls back to the default 0.5
```

## The `source` command

`source` runs a script **in the current shell process** instead of spawning a new child process. This means any variables or PATH changes the script makes take effect in your current shell.

A familiar example is activating a Python virtual environment:

```bash
source .venv/bin/activate
```

The activation script needs to modify `PATH` (and a few other variables) **in your current shell** so that `python` and `pip` resolve to the venv's binaries. If you ran it as a regular script (`bash .venv/bin/activate`), those changes would happen inside a child process and disappear the moment it exits - your shell would remain unchanged and the venv would not be active.

You can verify this with the `$$` variable, which holds the current process ID:

```console
myuser@hostname:~$ echo $$
44132
myuser@hostname:~$ bash .venv/bin/activate   # runs in a new process - has no effect
myuser@hostname:~$ echo $$
44132
myuser@hostname:~$ source .venv/bin/activate  # runs in THIS shell - venv is now active
(.venv) myuser@hostname:~$ echo $$
44132
```

Same PID throughout - `source` never left the current shell.

# Exercises

### :pencil2: Create your own Linux "command"

Let's create a shell program and add it to your `$PATH` env var. Execute the following commands line by line:

1. In your home dir, create a directory called `scripts`. This dir will be added to the PATH soon.
2. Create bash script in a file called `myscript` (without any extension), with the following content:

```bash
#!/bin/bash
echo my script is running...
```

3. Test your script by `bash myscript`
4. Give it execute permissions
5. Copy your script into `~/scripts`
6. Add `~/scripts` to the PATH (don't override the existing content of PATH, take a look at the above example).
7. Test your new "command" by just typing `myscript`.
8. Try to use the `myscript` command in another new terminal session. Does it work? Why?


