# Shells


A **shell** is the program that reads the commands you type and runs them. When you open a terminal, you are inside a shell. The shell sits between you and the Linux kernel - it translates your instructions into actions the operating system can execute.

The most common shell on Linux is **Bash** (Bourne-Again SHell). You will use it constantly as a developer or system administrator.

## From a list of commands to a real script

The best way to understand scripting is to watch a simple idea grow into a proper program. Consider a task: **clear old log files in `/var/log`**.

**Version 1 - just commands pasted into a file:**

```bash
cd /var/log
cat /dev/null > messages
cat /dev/null > wtmp
echo "Log files cleaned up."
```

This works, but it has some serious issues:

- It assumes you execute the script as root. 
- If some command fails, bash by default will continue running the next command, finally printing "Logs cleaned up" even though it failed.


**Version 2 - production-quality with error handling:**

```bash
#!/bin/bash
LOG_DIR=/var/log
ROOT_UID=0       # Only users with $UID 0 have root privileges

E_XCD=86         # Exit code: can't change directory
E_NOTROOT=87     # Exit code: not root

# Require root
if [ "$UID" -ne "$ROOT_UID" ]; then
  echo "Must be root to run this script."
  exit $E_NOTROOT
fi

# Verify we are in the right directory before modifying anything
cd $LOG_DIR
if [ "$PWD" != "$LOG_DIR" ]; then
  echo "Can't change to $LOG_DIR."
  exit $E_XCD
fi

cat /dev/null > messages
cat /dev/null > wtmp

echo "Log files cleaned up."
exit 0
```

## Shell types

Linux ships with several shells. The two you will encounter most often:

| Shell | Description |
|---|---|
| `sh` | The original Bourne Shell. Minimal and available everywhere. |
| `bash` | The Bourne-Again Shell. A superset of `sh` with many extras. The default shell on most Linux systems. |

To see all shells installed on your system:

```bash
cat /etc/shells
```

To find the default shell for your user account (it is stored in `/etc/passwd`):

```bash
echo $USER
cat /etc/passwd | grep $USER
```

The last field on your line is your default shell - typically `/bin/bash`.

### Restricted Bash (`rbash`)

`rbash` is a locked-down version of Bash that prevents users from doing things like changing directories or setting the `PATH`. It is sometimes assigned to restricted accounts:

```bash
rbash          # start a restricted bash session
cd /var        # try to change directory - what happens?
```

You should see a "restricted" error. This is intentional - `rbash` limits what a user can do.

## Running a script

There are two ways to run a Bash script:

**Method 1: as an executable**
```bash
chmod +x myscript.sh   # mark the file as executable (only needed once)
./myscript.sh          # run it
```
The `./` is required because the current directory is not in `$PATH` by default.

**Method 2: pass it to bash directly**
```bash
bash myscript.sh
```
This works even without `chmod +x`. The shebang line is ignored - bash is used regardless.

## The shebang line

When you run a script with `./myscript.sh`, the OS reads the first line to decide which interpreter to use. That line is the **shebang**:

```
#!/bin/bash
```

The `#!` is the magic marker, and `/bin/bash` is the path to the interpreter. If this path is wrong, you will get a "Command not found" error when running the script.

> **What happens without a shebang?** The system falls back to `/bin/sh`, which may behave differently from Bash. Always include the shebang to be explicit.


## Shell contexts: login, non-login, interactive, non-interactive

Not all shells are created equal. Bash behaves slightly differently depending on *how* it was started. There are two independent dimensions:

**Login vs. non-login:**
- A **login shell** starts when you first authenticate to a shell session, e.g. SSH or `su -l username`. 
- A **non-login shell** starts inside an existing session, e.g. a new terminal tab, or a shell spawned from a `bash` command.

**Interactive vs. non-interactive:**
- An **interactive shell** reads commands from your keyboard and shows a prompt.
- A **non-interactive shell** runs a script and exits - no prompt, no user input.

These two dimensions combine: a script you run with `bash myscript.sh` is **non-login and non-interactive**. An SSH session is **login and interactive**.

> #### 🧐 Think it through
>
> Look at this terminal session and classify each numbered line:
>
> ```bash
> 1   myuser@host:~$ su -l john    # switch to john with a login shell
>     john@host:~$
> 2   john@host:~$ rbash           # start restricted bash
> 3   john@host:~$ sh -c 'echo hi' # run a one-off command in sh
> ```
>
> For each of lines 1, 2, and 3: is the new shell **login or non-login**? **Interactive or non-interactive**?
>
> <details>
> <summary>Hints</summary>
>
> - `su -l` (the `-l` flag) simulates a full login.
> - `rbash` started from within an existing session is not a login shell.
> - `sh -c '...'` runs a command and exits - the user never types into it.
> </details>

## Configuration files: who runs what, and when

Bash reads different configuration files depending on the shell context. This is how your `PATH`, aliases, and environment variables get set up automatically.

### System-wide files (affect all users)

| File | When it runs |
|---|---|
| `/etc/profile` | Login shells only. Sets system-wide environment variables and `PATH`. |
| `/etc/profile.d/*.sh` | Sourced by `/etc/profile`. Used by packages to add their own settings. |
| `/etc/bash.bashrc` | Non-login interactive shells. Sets system-wide Bash options. |

### Per-user files (in your home directory)

| File | When it runs |
|---|---|
| `~/.bash_profile` | Login shells. Your personal environment setup - `PATH` additions, etc. |
| `~/.bashrc` | Non-login interactive shells. Aliases, functions, prompt customizations. |
| `~/.profile` | Login shells, if `~/.bash_profile` does not exist. Also read by `sh`. |

**The key rule:** if you open a new terminal tab in a desktop environment, you get a non-login interactive shell - so `~/.bashrc` is what runs. If you SSH in, you get a login interactive shell - so `~/.bash_profile` (or `~/.profile`) runs.

> #### 🧐 Explore your own config
>
> Run `ll` in your terminal. If it works, it is probably an alias for `ls -l` defined in `~/.bashrc`.
>
> Open `~/.bashrc` in a text editor and find the alias definition:
> ```bash
> nano ~/.bashrc
> ```
> Then add an alias of your own - something you will actually use. Reload the file without restarting the terminal:
> ```bash
> source ~/.bashrc
> ```
> Test that your new alias works.

---

# Exercises


### :pencil2: The nobody user - what shell does a system account use?

System accounts like `nobody` exist for security purposes, not for human logins. Let's find out what shell they are assigned.

1. Look up the `nobody` user in `/etc/passwd`:
   ```bash
   grep nobody /etc/passwd
   ```
2. What is the shell listed for `nobody`? Why do you think that shell was chosen?
3. As root, try to switch to the `nobody` user: `sudo su -l nobody`. What happens? What does the error or output tell you about that shell?


