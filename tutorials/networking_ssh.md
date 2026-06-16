# SSH Protocol 

The SSH protocol (Secure Shell) is a widely used protocol for secure remote access to systems and devices.
It provides strong encryption and authentication mechanisms, allowing users to securely log in to remote machines and execute commands or transfer files.

In addition to traditional password authentication, the Secure Shell application can use public key cryptography to authenticate users.

## Basic usage

The basic usage of the `ssh` command is as follows:

```shell
ssh [user@]hostname [command]
```

- `user`: The username to use for the remote connection (optional).
- `hostname`: The hostname or IP address of the remote system.
- `command`: The command to execute on the remote system (optional). If you omit the command, ssh will open a shell session on the remote system, allowing you to interact with it as if you were physically sitting in front of it.

If you are connecting to the system for the first time, ssh will ask you to verify the authenticity of the remote host by displaying the fingerprint of the remote system's public key.

```console
myuser@hostname:~$ ssh ec2-user@server1
The authenticity of host 'server1 (192.168.0.254)' can't be established.
RSA key fingerprint is fc:c8:87:90:f0:39:af:4f:de:99:cc:30:ce:64:b2:8e.
Are you sure you want to continue connecting (yes/no)?
```

Once you have verified the fingerprint, ssh will establish a secure connection to the remote system and open a shell session.
Verified server's fingerprint are stored in the clients machine under `~/.ssh/known_hosts`.

On subsequent connections to the same server, the SSH client will check the fingerprint of the server's public key against the fingerprints stored in the `known_hosts` file.
If the fingerprint of the server's public key has changed, the SSH client will display a warning message and refuse to connect to the server.
This helps to protect against man-in-the-middle attacks.


## Generating a public-private key-pair locally

When using ssh, a user's public-private key pair can be generated with the `ssh-keygen` command.

```console
myuser@hostname:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/myuser/.ssh/id_rsa): ENTER
Enter passphrase (empty for no passphrase): ENTER
Enter same passphrase again: ENTER
Your identification has been saved in /home/myuser/.ssh/id_rsa.
Your public key has been saved in /home/myuser/.ssh/id_rsa.pub.
The key fingerprint is:
e0:71:43:df:ed:40:01:0b:44:54:db:c2:80:f2:33:aa myuser@station
```

The user `myuser` was first prompted for the new (private) key's filename, to which `myuser` simply hit ENTER to accept the default filename: `~/.ssh/id_rsa`. 
**Make sure that you don't override important existed key when you are playing with `ssh-keygen`.**

Next, `myuser` was given the opportunity to attach a passphrase to his private key.
By hitting ENTER again (twice), `myuser` chose not to. 

The private key was generated under `/home/myuser/.ssh/id_rsa`,
while the public key under `/home/myuser/.ssh/id_rsa.pub`.

You should **NEVER** send your private key, not even on a secure channel. 

## Allowing account access: `~/.ssh/authorized_keys`

SSH access to a server is granted by obtaining a copy of the public key of the person who is to be granted access, and storing it in the server's `~/.ssh/authorized_keys` file.
More people can be granted access to an account by simply appending their public keys to the `~/.ssh/authorized_keys` files, one public key per line.

The `~/.ssh/authorized_keys` file, and the whole `~/.ssh` directory, must only be readable by the user.
How the copy of the public key is obtained does not matter. It could be emailed, scped (as discussed in a moment), or transferred from one terminal to another using the mouse's cut and paste buffer.


## Transferring files securely and easily: `scp`

The `scp` command uses a syntax almost identical to the `cp` command, but either the source file(s) or the
destination file can be on a remote machine. 

When referring to a file on a remote machine, the following syntax is used:

```shell
user@host:path
```

As an example, the following command line would transfer the `/etc/services` file from `server1` into the `~/cfg/server1/etc/` directory in `myuser`'s home directory:

```console
myuser@hostname:~$ scp ec2-user@server1:/etc/services cfg/server1/etc/
services    100% |*****************************| 19936      00:00
```

The `-r` command line switch (for "recursive") must be specified when copying an entire directory (and its subdirectories).
In the following, `myuser` recursively copies the `/etc/sysconfig` directory from his local machine to the machine `server1`'s `/tmp` directory:

```console
myuser@hostname:~$ scp -r /etc/sysconfig ec2-user@server1:/tmp
ifup-aliases    100% |*****************************| 13137  00:00
ifcfg-lo        100% |*****************************| 254    00:00
ifdown          100% |*****************************| 3676   00:00
ifdown-ippp     100% |*****************************| 820    00:00
ifdown-ipv6     100% |*****************************| 4076   00:00
```

# Exercises


### :pencil2: Port forward

SSH port forwarding (also known as SSH tunneling) allows you to securely tunnel traffic between a local and a remote machine over an encrypted SSH connection. 
This enables you to access remote services as if they were running on your local machine.

In this exercise, we'll use port forwarding to access a Python web server running on the EC2 instance:

1. First, make sure the `simple_python_server` is running in your instance (you can leverage your existed instance with the app running as a linux service).
2. Connect to the instance via `ssh`, while adding the following flag: `-L 8080:localhost:8080` to the `ssh` command. 

> [!NOTE]
> If port `8080` is already bounded to another process in your local machine, you can forward port `8081` by `-L 8081:localhost:8080`


### :pencil2: Add new keys

Allow to connect to the instance using another keys.

1. Generate another RSA key-pair using the `ssh-keygen` command
2. Add the generated public key to the `~/.ssh/authorized_keys` file in the remote machine.
3. Close the existed SSH connection and try to connect using the new identity by:

```bash
ssh -i <new_private_key> ubuntu@<public-ip>
```

### :pencil2: Change the ssh daemon

Change the default SSH port (22) to a custom port on your EC2 instance:

1. In your EC2, edit the SSH daemon configuration file:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
2. Find the line that says `#Port 22` and change it to `Port 2222`.
3. Save the file and restart the SSH daemon by `sudo systemctl restart sshd`.
4. Update the security group rules to allow inbound traffic on port 2222.
5. Open a new terminal and test connecting using the new port:
   ```bash
   ssh -i <key-pair> -p 2222 ubuntu@<public-ip>
   ```

> [!WARNING]
> Do not close your existing SSH session until you confirm you can connect on the new port. This prevents getting locked out if there are any configuration issues.
