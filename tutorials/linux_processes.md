# Linux Processes

## Process Definition

A process is a currently executing program (or command). A program is a series of instructions that tell the computer what to do. When we run a program, those instructions are copied into memory and space is allocated for variables and other stuff required to manage its execution. This running instance of a program is called a **process**.

The top command can show running processes in the terminal in real-time.

```console
myuser@hostname:~$ top
...
```

## Processes state

In linux, multiple users running multiple processes, at the same time and on the same system. The CPU manages all these processes simultaneously, according to the below state diagram:

![][linux_process_state]

A process can be in one of several states at any given time, indicating its current status or activity. These states are important to understand for managing and troubleshooting system performance.

Here are a short description of each state:

1. **Running**: A process is currently executing on a CPU core (a.k.a process CPU burst).
2. **Waiting**: A process is waiting for some external event, such as user input, disk, or a network I/O operation to complete.
3. **Ready**: Is often used to refer to a process that is waiting to be executed on a CPU. When a process is in the ready state, it is typically placed in a queue, waiting for an available CPU to execute on. Once the CPU becomes available, the process is moved from the ready state to the running state and starts executing. The amount of time a process spends in the ready state is dependent on the scheduling algorithm and the current system load.
4. **Zombie**: A process has completed its execution but its parent process has not yet collected its exit status.
5. **Initialized**: A process that has been created but has not yet been assigned a process identifier (PID)
6. **Terminated**: Indicates that a process has finished its execution and has exited.

## The process tree

A new process is created because an existing one makes an exact copy of itself (forking). This implies a tree structure of processes:

```console
myuser@hostname:~$ pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager───2*[{NetworkManager}]
        ├─2*[SACSrv───3*[{SACSrv}]]
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─acpid
        ├─atd
        ├─avahi-daemon───avahi-daemon
.
.
.
```

This child process has the same environment as its parent, only different process id (PID). When a process ends normally (it is not killed or otherwise unexpectedly interrupted), the program returns its **exit code** (a number) to the parent. Only exit status of 0 means success.

A process has a series of characteristics, which can be viewed with the `ps` command:

```console
myuser@hostname:~$ ps -aux
USER     	PID %CPU %MEM	VSZ   RSS TTY  	STAT START   TIME COMMAND
root       	1  0.0  0.1 172752  9440 ?    	Ss   13 Feb  7:21 /sbin/init splash
root       	2  0.0  0.0  	0 	0 ?    	S	13 Feb  0:00 [kthreadd]
```

The `ps` command can display a lot of information about processes, including their PID, state (STAT column), CPU and memory usage, the user owning this process, and the command that initiated the process.


## Signals 

Processes are communicating with each other and with the kernel using **signals**. There are multiple signals that you can send to a process. Use the `kill` command to send a signal to a process.

| Signal Name | Signal Number | Meaning                                                     |
|-------------|---------------|-------------------------------------------------------------|
| SIGTERM     | 15            | Terminate the process in orderly way                        |
| SIGINT      | 2             | Interrupt the process. The process can ignore this signal   |
| SIGKILL     | 9             | Interrupt the process. The process can't ignore this signal |
| SIGHUP      | 1             | The parent process was terminated                           |


Use `man 7 signal` for a comprehensive list of linux signals.

Let's kill a process by send it a KILL signal:

```console
myuser@hostname:~$ sleep 600
```

The above command initiates a process that just sleeps 600 seconds, and ends.

In **another terminal**, use the `ps` to get the PID of the process running the `sleep` command:

```console
myuser@hostname:~$ ps -a
PID TTY          TIME CMD
46214 pts/1    00:00:00 sleep
46445 pts/3    00:00:00 ps
...
myuser@hostname:~$ kill -9 46214
```

Observe how the process running the `sleep` command is terminated.

---

In bash, you can use keyboard shortcuts to send signals to process:

| Shortcut | Signal  | Meaning                               |
|----------|---------|---------------------------------------|
| CTRL+C   | SIGINT  | Terminate the process in orderly way  |
| CTRL+Z   | SIGSTOP | Suspend the process in the background |


### Graceful termination of processes

We now demonstrate a very important concept called **Graceful Termination**.

Graceful termination refers to the process of shutting down a program (a process) in a way that allows it to complete its current tasks and close down all processes in a safe and controlled manner. This ensures that no data is lost, and all system resources are properly released.

Let's implement a graceful termination logic for the Yolo app.

Add the following to `app.py`:

```python
import signal
import sys

is_shutting_down = False

def handle_sigterm(signum, frame):
    global is_shutting_down
    is_shutting_down = True
    logging.info("Received SIGTERM. Shutting down gracefully...")
    # Perform cleanup: close DB connections, finish pending work, etc.
    logging.info("Cleanup done. Exiting.")
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

And add a `/ready` endpoint that returns an error while the app is shutting down:

```python
@app.get("/ready")
def ready():
    if is_shutting_down:
        raise HTTPException(status_code=503, detail="Service is shutting down")
    return {"status": "ready"}
```

When the process receives `SIGTERM`, Python invokes `handle_sigterm` instead of terminating immediately. The `is_shutting_down` flag is set to `True` first, so any new requests to `/ready` immediately get a `503` response - signalling to load balancers or orchestrators (like Kubernetes) to stop routing traffic to this instance. The app then performs cleanup and exits cleanly.

With this in place, send `SIGTERM` to the running server process:

```console
myuser@hostname:~$ kill -15 <PID>
```

Quickly call `/ready` right after - you should get a `503`. Compare this to sending `SIGKILL` (`kill -9`), which terminates the process immediately - the handler is never called and no cleanup runs.

## Services

In Linux, **services** are background processes that run continuously and provide specific functions to the OS or other applications - `ssh`, `cron`, `ufw`, and `apache` are common examples.

Services are managed by **systemd**, the first process started by the kernel (PID 1). Systemd reads **unit files** - plain-text configuration files that describe how to start, stop, and restart a process. Unit files live in `/etc/systemd/system/`.

Use `systemctl` to manage services:

```console
myuser@hostname:~$ sudo systemctl status ufw
myuser@hostname:~$ sudo systemctl start ufw
myuser@hostname:~$ sudo systemctl stop ufw
myuser@hostname:~$ sudo systemctl restart ufw
myuser@hostname:~$ sudo systemctl enable ufw   # start automatically on boot
```

List all services:

```console
myuser@hostname:~$ systemctl list-units --type=service
```

# Exercises

### :pencil2: Run the YOLO app as a Linux service

In this exercise you will register the YOLO detection app as a systemd service.

1. Create a unit file at `/etc/systemd/system/yolo.service`:

```text
[Unit]
Description=YOLO Detection App
After=network.target

[Service]
WorkingDirectory=/path/to/your/yolo/app
ExecStart=/path/to/your/.venv/bin/python app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- `WorkingDirectory` - the directory containing `app.py`.
- `ExecStart` - the full path to the Python interpreter inside your venv, followed by `app.py`.
- `Restart=on-failure` - systemd automatically restarts the process if it crashes.

Adjust the paths to match your setup.

2. Reload systemd so it picks up the new file:
   ```bash
   sudo systemctl daemon-reload
   ```
3. Start the service: `sudo systemctl start yolo.service`
4. Check its status: `sudo systemctl status yolo.service`
5. View live logs: `journalctl -fu yolo.service`
6. Send `SIGTERM` to the service process and observe the graceful shutdown logs: `sudo systemctl stop yolo.service`

**Important - clean up when done.**  
Now that you've seen how it works, stop and remove the service. Leaving it running will conflict with the app when you run it manually during development:

```bash
sudo systemctl stop yolo.service
sudo systemctl disable yolo.service
sudo rm /etc/systemd/system/yolo.service
sudo systemctl daemon-reload
```


### :pencil2: Run processes in the background terminal session

"Interactive processes" are processes that are initialized and controlled through a terminal session.
These processes can run in the foreground, occupying the terminal. 
Alternatively, they can run in the background. The terminal can accept new commands while the program is running.

1. Open a new terminal session and execute `sleep 600` which initiates a process that "sleeps" 10 minutes and ends.
2. Stop the process and send it into the background by `CTRL+Z`.
3. Bring the process to the foreground by `fg`.
4. Now run the same command while sending the process to the background by the `&` operator: `sleep 600 &`.
5. Bring the process to the foreground.
6. Kill the process by `CTRL+C`.


[linux_process_state]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/linux_process_state.png