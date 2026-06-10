# Bash Command Techniques

## Exit status and `$?`

Every command you run returns a number to the shell when it finishes. This number is called the **exit status** (or return code):

- `0` means the command succeeded
- Any non-zero value means something went wrong

You can always check the exit status of the last command with `$?`:

```bash
ls /non-existing-dir
echo $?          # prints 2 - the directory doesn't exist
```

```bash
ls /etc
echo $?          # prints 0 - success
```

The `$?` variable is overwritten after every command, so you need to capture it immediately if you want to use it later.

Common exit status values:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General error |
| `2` | Misuse of a shell built-in (wrong arguments, etc.) |
| `126` | Command found but not executable |
| `127` | Command not found |
| `128+N` | Process killed by signal N |


## Running multiple commands

### Unconditional - `;`

A semicolon runs commands one after another, regardless of whether the previous one succeeded:

```bash
cd /tmp; ls
```

The `ls` runs even if `cd` fails. This is rarely what you want when one command depends on the other.

### `&&` - run the second only if the first succeeded

`&&` connects two commands so the second only runs if the first exits with `0`:

```bash
mkdir /tmp/myproject && mv report.txt /tmp/myproject
```

If `mkdir` fails (the directory already exists, or you lack permissions), `mv` is skipped. This prevents you from accidentally moving a file to the wrong place.

### `||` - run the second only if the first failed

`||` is the opposite: the second command runs only if the first exits non-zero:

```bash
chmod 600 /tmp/myproject/report.txt || echo "Could not set permissions."
```

This is useful as a quick fallback or error message. If `chmod` succeeds, the `echo` is skipped.

### Combining them

You can chain `&&` and `||` together to express simple success/failure logic in one line:

```bash
mkdir /tmp/logs && echo "Directory created." || echo "Failed to create directory."
```

This reads: try `mkdir`; if it succeeds, print the success message; if it fails, print the failure message.

> **Note:** The `||` at the end applies to the result of the `&&` expression, so if `mkdir` succeeds but the `echo` somehow fails, the failure message would still print. For scripts that need to be robust, use `if` statements instead.

## Command substitution

Command substitution lets you use the *output* of a command as part of another command. The syntax is:

```bash
$(command)
```

For example, to create a directory named with today's date:

```bash
mkdir reports.$(date +%d%b%Y)
ls
# reports.10Jun2026
```

Bash runs `date +%d%b%Y` in a subshell, captures its output, and substitutes it in place of `$(...)` before running `mkdir`.

Another practical example - store the current user in a variable:

```bash
owner=$(whoami)
echo "Running as: $owner"
```

Command substitution works anywhere a string is expected: in variable assignments, command arguments, filenames, and more.


