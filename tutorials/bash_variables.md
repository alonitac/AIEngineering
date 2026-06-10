# Bash Variables

A variable is a name that holds a value. Variables let you store data, avoid repeating yourself, and write scripts that adapt to different inputs.

In Bash, you assign a variable like this:

```bash
name=Alice
```

And you read it back by prefixing the name with `$`:

```bash
echo $name    # Alice
```

**Important:** there must be no spaces around the `=` sign. These all fail:

```bash
name = Alice   # error: "name" is interpreted as a command
name =Alice    # error
name= Alice    # error
```

## Assigning and referencing variables

```bash
greeting=Hello
echo $greeting        # Hello
echo ${greeting}      # Hello - curly braces make the boundary explicit
echo "$greeting"      # Hello - double quotes allow variable expansion
echo '$greeting'      # $greeting - single quotes suppress expansion
```

The curly brace form `${VAR}` matters when the variable name is adjacent to other characters:

```bash
A=hola
echo "$A_world"    # prints nothing - bash looks for variable A_world (undefined)
echo "${A}_world"  # hola_world - bash expands A, then appends _world
```

This is a common bug. If a variable expansion produces an unexpected empty string, check whether you need `${}`.

When you are done with a variable, you can unset it:

```bash
unset A
echo $A    # prints nothing
```

## Variables are untyped - but context matters

Bash does not have types like integers or strings. Every variable is stored as a string. But Bash is smart enough to do arithmetic when the context requires it:

```bash
a=5
echo $a          # 5 (as a string)
echo $a+3        # 5+3 (no arithmetic - just string concatenation)

let a=5+3
echo $a          # 8 (let performs integer arithmetic)
```

If you need arithmetic, use `let` or the `$(( ))` syntax (covered more in the arithmetic tutorial):

```bash
result=$((10 + 4))
echo $result     # 14
```

## Storing command output in a variable

Use `$(...)` to capture the output of a command into a variable:

```bash
today=$(date +%Y-%m-%d)
echo $today      # 2026-06-10

arch=$(uname -m)
echo $arch       # x86_64 (or whatever your architecture is)
```

This is called **command substitution**. The command inside `$(...)` runs in a subshell and its output replaces the expression.

## Positional parameters

When you run a script, anything you type after the script name is available inside the script as `$1`, `$2`, `$3`, etc.

Create a file called `greet.sh`:

```bash
#!/bin/bash
echo "Hello, $1! You passed $# argument(s)."
echo "All arguments: $@"
```

Run it:

```bash
bash greet.sh Alice
# Hello, Alice! You passed 1 argument(s).
# All arguments: Alice

bash greet.sh Alice Bob Carol
# Hello, Alice! You passed 3 argument(s).
# All arguments: Alice Bob Carol
```

### Special variables at a glance

| Variable | What it contains |
|---|---|
| `$0` | The name of the script itself |
| `$1`, `$2`, ... | The 1st, 2nd, ... argument |
| `$#` | The number of arguments passed |
| `$@` | All arguments as separate words |
| `$*` | All arguments as a single string |
| `$?` | Exit status of the last command |
| `$$` | Process ID of the current shell |

> For arguments beyond `$9`, wrap in braces: `${10}`, `${11}`, etc.

## Variable expansion tricks

Bash has built-in ways to manipulate variables when you reference them.

### Default value - `${VAR:-default}`

If `VAR` is unset or empty, use `default` instead:

```bash
echo ${NAME:-"stranger"}   # prints "stranger" if NAME is not set
NAME=Alice
echo ${NAME:-"stranger"}   # prints "Alice"
```

This is handy for making scripts safe when an argument is missing.

### Error if unset - `${VAR:?message}`

If `VAR` is unset or empty, print `message` to stderr and exit:

```bash
echo ${REQUIRED_VAR:?"REQUIRED_VAR must be set"}
```

Use this to catch missing configuration early rather than letting a script fail in a confusing way later.

### Substring - `${VAR:offset:length}`

Extract part of a string:

```bash
s="Hello, World"
echo ${s:7}      # World
echo ${s:7:5}    # World
echo ${s:0:5}    # Hello
```

### String length - `${#VAR}`

```bash
s="Hello"
echo ${#s}    # 5
```


# Exercises

### :pencil2: Variable basics - spot the bugs

Each snippet below has a problem. Identify what is wrong, fix it, and explain why your fix works.

1. ```bash
   city = London
   echo "I live in $city"
   ```

2. ```bash
   prefix=backup
   touch $prefix_2026.txt    # intended filename: backup_2026.txt
   ```

3. ```bash
   greeting='Hello, $USER'
   echo $greeting            # intended: Hello, alice
   ```

### :pencil2: Dated file copy

Write a script called `datedcp.sh` that takes a filename as its first argument and creates a copy of the file with today's date prepended to the name.

```bash
bash datedcp.sh report.txt
ls
# 2026-06-10_report.txt   report.txt
```

Requirements:
- Use `$(date +%Y-%m-%d)` for the date
- Print an error and exit if no argument is given
- Print an error and exit if the file does not exist (hint: use `[ -f "$1" ]`)

