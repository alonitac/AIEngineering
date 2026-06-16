# Networking Security 

## Motivation 

![][networking_alice_bob]

Alice, Bob, and Eve are often used in cryptography to represent different entities with varying levels of trust.
Alice and Bob are usually considered trusted parties who want to communicate securely,
while Eve is an eavesdropper who wants to intercept their communication. Eve can **see and change** every message that is being sent over the channel. 

For example, Alice may represent a bank company website, while Bob is a bank customer who wants to perform some actions on his online account. 
If the communication channel is not secured (the data is sent as a plain text), Eve, as an eavesdropper, can capture the login credentials of Bob, or, change the messages that Bob sends to the Bank (e.g. send 100$ to ~~John~~ Eve's account). 

In this tutorial, we will be learning the fundamentals of network security and encryption.

## Encryption 

When Julius Caesar sent messages to his generals, he didn't trust his messengers. So he replaced every A in his messages with a D, every B with an E, and so on through the alphabet. Only someone who knew the “shift by 3” rule could decipher his messages.

And so we begin…

**Cryptography** is the science of using mathematics to encrypt and decrypt data.

A cryptographic algorithm, or **cipher**, is a mathematical function used in the encryption and decryption process.
A cryptographic algorithm works in combination with a key -a word, number, or phrase- to encrypt the plaintext.
The same plaintext encrypts to different ciphertext with different keys.

## Symmetric key encryption

Symmetric key encryption is a method of cryptography in which the same secret key is used for both encryption and decryption.

![][networking_symmetric-key-enc]

We'll use `openssl` to encrypt messages:

```console
myuser@hostname:~$ echo 'vv7:K0r|E[PC!JM' > .secretKey
myuser@hostname:~$ echo "I'm Bob, want to transfer 500$ to John using the following credit card number 1234 1234 1234 1234" > message.txt
myuser@hostname:~$ openssl enc -e -aes-256-cbc -salt -in message.txt -out encrypted_message.txt -pass file:.secretKey
myuser@hostname:~$ cat encrypted_message.txt
Salted__�ǝjo7....
```

The above example encrypts the content of `message.txt` using a key stored under `.secret`. 
The resulted output was create under `encrypted_message.txt`.
This file can be sent safely over any channel, since it contains Gibberish for a person who doesn't know the secret key. 

The above `openssl` command was executed with the following flags: 

- `-d`              Decrypt file
- `-aes-256-cbc`    The symmetric encryption algo name
- `-salt`           Adds a salt to the key to make it more secure.
- `-k`              User password
- `-in`             The input file
- `-out`            The output file
- `-pass`           Specifies the path to the file containing the key used for encryption (.secret in out case)

Read [openssl docs](https://www.openssl.org/docs/manmaster/man1/openssl-enc.html) for more information.  

We now want decrypt the encrypted file:

```console
myuser@hostname:~$ openssl enc -d -aes-256-cbc -salt -in encrypted_message.txt -out original_message.txt -pass file:.secretKey
myuser@hostname:~$ cat original_message.txt
I'm Bob, want to tr...
```

## Asymmetric key encryption

In the previous section we've seen how client and server can communicate using a shared key.
But keep in mind that this key must be agreed by both sides first. **How can both parties agree on a key if the only communication channel they have is unsecure?**
Can you come up with a way that both parties can securely exchange the secret with which they will encrypt and decrypt messages? 

We can achieve it using Asymmetric Encryption.

Asymmetric key encryption, also known as **public key cryptography**, uses a pair of related keys to encrypt and decrypt data.
One key, known as the **public key**, is shared with anyone who wants to send encrypted data to the owner of the key.
The other key, known as the **private key**, is kept secret and used to decrypt data that has been encrypted with the public key.

![][networking_asymmetric-key-enc]

Let's generate a public-private key pair and encrypt messages. We will use `openssl` again.


Generate a `1024` bit length private key.
Choose an easy pass phrase for your key, so you can remember it later.

```
openssl genrsa -aes256 -out private.key 1024
```

Take a look at your private key.

Using the generated private key, generate the public key pair:

```
openssl rsa -in private.key -pubout -out public.key
```

Take a look at your private key.

Encrypt a message by:
```
openssl rsautl -encrypt -pubin -inkey public.key -in message.txt -out encrypted_message.txt
```

Can you see that only the public key has used to encrypt the message, and since public keys are not secrets, everyone can encrypt messages and send them to the owner of the public key, so he could decrypt them using his private key.

Let's decrypt the file by:
```
openssl rsautl -decrypt -inkey private.key -in encrypted_message.txt
```

## Digital signature

Take a closer look on the below encryption scheme.
Now Alice is sending a message to Bob. She encrypts the message using her **private key**, and Bob decrypts it using Alice's **public key**.

What is wrong here?

![][networking_digital-signature]

This scheme has no meaning in terms of encryption! But it can be a useful technique to verify the **authenticity and integrity** of a message.

Remind yourself that Eve is not only able to read the transferred messages between Bob and Alice, but also to **edit** any message.
And now the parties have another concern other than privacy: how can Bob know that incoming messages from Alice are authentic? That they were not altered by Eve? 

The private key is used to **sign** the data and the recipient uses the sender's public key to verify the digital signature.

We will use the above key-pair to sign a message:

```shell
openssl dgst -sha256 -sign private.key -out signature.txt message.txt
```

Now `signature.txt` is your signature for the message `message.txt`. Alice sends **both** `message.txt` and `signature.txt` to Bob (we don't care privacy here, only authenticity!).

Bob now verifies that the message he got, `message.txt`, indeed written by Alice:

```shell
openssl dgst -sha256 -verify public.key -signature signature.txt message.txt
```

## Digital certificates

A **certificate**, simple put, is the server's public key, accompanied by some other attributes such as Organization name, Locality, Country name etc...

The certificate is **digitally signed by Certificate Authority** (CA), an entity that stores, signs, and issues digital certificates.
CA acts as a trusted 3rd party both by the client and the server.

Common CA are [DigiCert](https://en.wikipedia.org/wiki/DigiCert), [Let's Encrypt](https://en.wikipedia.org/wiki/Let%27s_Encrypt), AWS and the public cloud providers.

Why do we need that? 

When Alice first sends her public key to Bob, there is real risk that Bob receive Eve's public key, but b
When Alice first sends her public key to Bob, Bob wants to be sure that the key actually belongs to Alice and has not been tampered with or replaced by Eve. 

In a typical scenario, Alice would generate a public-private key pair and send her public key to a trusted CA along with some identifying information, such as her name or email address.
The CA would then verify Alice's identity and issue a digital certificate that contains Alice's public key and identifying information, as well as the CA's own digital signature. 
This certificate serves as proof that Alice's public key belongs to her and has not been tampered with.

Bob can then verify the certificate using the CA's public key (which is typically included in his web browser or operating system) and be reasonably sure that the key actually belongs to Alice and has not been tampered with.

## Putting it all together - HTTPS protocol and the TLS Handshake 

**Although we will be focusing on the HTTPS and TLS protocols, the discussed security techniques are widely used in many other protocols and systems.**

HTTPS (HyperText Transfer Protocol Secure) is an encrypted version of the HTTP protocol usually running on **port 443**.
It uses SSL or TLS protocol to encrypt all communication between a client and a server.
This secure connection allows clients to safely exchange sensitive data with a server, such as when performing banking activities or online shopping.

Consider the above mentioned banking scenario: Bob sends to its Bank website (represented by Alice) an HTTP request which essentially says "I'm Bob, want to transfer 500$ to John using the following credit card number....".
If no security measures are taken, Bob could be surprised in a few ways, as detailed below:

- If no **encryption** is used, an intruder (man-in-the-middle) could intercept Bob's request and use Bob's credit card details.
- If no **data integrity** is used, an intruder could modify Bob's transaction (e.g. change "John" to "Eve").
- Finally, if no **server authentication** is used, Bob may believe that that server he is talking with, is the original Bank website, while actually he talks with a faked website maintained by Eve, who is impersonating as the bank website. After receiving Bob's transaction, Eve could take and use Bob's information.

We now want to build security mechanism that provide: Privacy (encryption), Data Integrity, and Authenticity.

TLS handshake is the initial process that occurs between a client and a server when establishing a secure connection using the Transport Layer Security (TLS) protocol. It involves a series of steps including encryption negotiation, authentication, and key exchange. As detailed below: 

1. **Client hello**: The "Client hello" message is a request to the server to begin a TLS handshake.
   This message includes the TLS version supported by the client, a random value known as the client random, and a list of supported cipher suites and compression methods.
2. **Server Hello**: The server responds with a Server Hello message, which includes the TLS version selected for the connection, a random value known as the server random, and the chosen cipher suite and compression method from the client's list.
3. **Certificate**: The server sends its digital certificate, which includes its public key and other identifying information, to the client.
4. **Client Key Exchange**: The client generates a secret key (called pre-master key), encrypts it using the server's public key from the certificate, and sends the encrypted pre-master key to the server.
5. **Finished**: Both the client and server exchange "Finished" messages to confirm that they have successfully completed the handshake process.
   These messages include a hash of all the previous handshake messages, encrypted using the secret key.
6. **Secure Communication**: Once the handshake is complete, the client and server use a shared session key to symmetrically encrypt and decrypt data transmitted between them.
   The session key(s) were derived from the pre-master key. 

This process is a **simplified version** of what called the *TLS Handshake*.

## Hash functions 

A hash function is a mathematical function that takes in data of any size and produces a fixed-size output.
A simple example of a hash function is the MD5 algorithm, which generates a 128-bit hash value.
`md5sum` is a Linux command line utility used to generate and verify MD5 checksums of files.

```console
myuser@hostname:~$ echo "hello world" | md5sum
6f5902ac237024bdd0c176cb93063dc4
myuser@hostname:~$ echo "let it be" | md5sum
90b1d29ca90dac03f168407e4cd8fc15
myuser@hostname:~$ cat 5GB_file | md5sum
b638d7f49a67fe9b4c982ca3bc70066d
```

The hash function is designed to be a one-way function, which means that it is easy to compute the hash of a given input, but it is **computationally infeasible** to compute the input given its hash.
Hash functions are **deterministic**, which means that the same input data will always produce the same hash output:

```console
myuser@hostname:~$ X=$(echo "hello world" | md5sum)
myuser@hostname:~$ Y=$(echo "hello world" | md5sum)
myuser@hostname:~$ test "$X" = "$Y"
myuser@hostname:~$ echo $?
0
```

The distribution of hashed values is designed to be **uniform**, which means that different input values should produce very different hash values.
Even small changes in the input should produce significant changes in the output hash value:

```console
myuser@hostname:~$ echo "hello world" | md5sum
6f5902ac237024bdd0c176cb93063dc4
myuser@hostname:~$ echo "hello worln" | md5sum
a10a0af8e9b486a707aaccb7a752d37e
```

However, it is possible for different input values to produce the same hash value, which is called a **hash collision**.

Hash functions are commonly used to store passwords securely. 
Instead of storing the actual passwords, the hash of the password is stored in the database. 
When a user attempts to log in, the password entered is hashed and compared to the hashed password in the database. 
If they match, the user is granted access.

# Exercises



### :pencil2: Self-signed Certificate - Enable HTTPS on the YOLO API

> [!NOTE]
> This exercise is for learning purposes only. Throughout the course the YOLO service is treated as an **internal** service — it runs inside a private network and is only accessed by other internal components, so plain HTTP is perfectly fine for it. In a later session you will see how public-facing traffic is terminated at a load balancer or reverse proxy, which is where TLS lives in practice.

In real life, a trusted authority (like Amazon, DigiCert) signs your certificates, giving them validity. But for internal or development usage, you can sign the certificate yourself - a **self-signed certificate**. Browsers and `curl` will warn about it, but the encryption is just as strong.

#### Step 1 - Generate a self-signed certificate

Inside the `services/yolo` directory of your repo, generate a private key and a certificate in one command:

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes
```

The `-nodes` flag skips the passphrase prompt so uvicorn can load the key without interactive input.
The program will ask you some identifiable information (country, organisation, Common Name, etc.). Fill in whatever values you like - this is a self-signed cert and no CA will verify them.

#### Step 2 - Start the YOLO app over HTTPS

The YOLO app uses **uvicorn** as its underlying server. Uvicorn can serve TLS directly by passing the certificate and key files:

```bash
uvicorn app:app --host 0.0.0.0 --port 8443 \
    --ssl-keyfile key.pem \
    --ssl-certfile cert.pem
```

> [!NOTE]
> Make sure port **8443** is open in your EC2 Security Group for inbound traffic (TCP, your IP or `0.0.0.0/0` for testing).

#### Step 3 - Test the HTTPS endpoint

From your **local machine**, call the health endpoint:

```bash
curl https://<ec2-public-ip>:8443/health
```

Because the certificate is self-signed, `curl` will refuse the connection with an SSL error - exactly the same warning a browser would show. Use `-k` (`--insecure`) to bypass certificate verification for testing:

```bash
curl -k https://<ec2-public-ip>:8443/health
```

You should get `{"status":"ok"}`.

> [!WARNING]
> **Stop the HTTPS server when you are done with this exercise.** Do not leave it running as your default way to start the YOLO service. The YOLO service is an **internal** service, in the future you'll see that it is never directly exposed to the internet, so it does not need TLS. 


### :pencil2: Authenticity verification

Under `signature_verification/` in our course repo, you are given 5 signatures and the corresponding messages. Determine which of the signatures are authentic.




### :pencil2: TLS Handshake 

![][networking_alice_bob]

As you may know, the communication in HTTP protocol is insecure, and since Eve is listening on the channel between you (Alice) and the web server (Bob), you are required to create a secure channel.
This is what HTTPS does, using the TLS protocol. 
The process of establishing a secure TLS connection involves several steps, known as TLS Handshake.

The TLS protocol uses a combination of **asymmetric** and **symmetric** encryption. Here is a **simplified** TLS handshake process:

#### Step 1 - Client Hello (Client -> Server).

First, the client sends a **Client Hello** message to the server.
The message includes:

- The client's TLS version.
- A list of supported ciphers.

#### Step 2 - Server Hello (Server -> Client)

The server replies with a **Server Hello**.
A Server Hello includes the following information:

- **Server Version** - a confirmation for the version specified in the client hello.
- **Session ID** - used to identify the current communication session between the server and the client.
- **Server digital certificate** - the certificate contains some details about the server, as well as a public key with which the client can encrypt messages to the server. The certificate itself is signed by Certificate Authority (CA).


#### Step 3 - Server Certificate Verification

Alice needs to verify that she's taking with Bob, the real Bob.
Why should she suspect? Since Eve is controlling every message that Bob sends to Alice,
Eve can impersonate Bob, talk with Alice on his behalf, without Alice to knowing that. 

Here the CA comes into the picture. CA is an entity (e.g. Amazon Web Services, Microsoft etc...) trusted by both sides (client and server) that issues and signs digital certificates, so the ownership of a public key can be easily verified.

In this step the client verifies the server's digital certificate. Which means, Alice verifies Bob's certificate.

#### Step 4 - Client-Server master-key exchange


Now, after Bob's certificate was verified successfully, the client and the server should agree on a **symmetric key** (called **master key**) with which they will communicate during the session.
The client generates a 32-bytes random master-key, encrypts it using the server's certificate and sends the encrypted message in the channel.

In addition to the encrypted master-key, the client sends a sample message to verify that the symmetric key encryption works.

> [!NOTE]
> The real TLS protocol doesn't use the master key for direct communication. Instead, other different session keys are generated and used to communicate symmetrically. 
> Both the client and the server's generate the keys each in his own machine.  

#### Step 5 - Server verification message

The server decrypts the encrypted master-key. 

From now on, every message between both sides will be symmetrically encrypted by the master-key.
The server encrypts the sample message and sends it back to the client.

#### Step 6 - Client verification message

The client verifies that the sample message was encrypted successfully.

### Implement the TLS handshake

Use the `scp` command to copy the directory `tls_webserver` into your home directory of your EC2 instance. 
This Python code implements an HTTP web server that represents Bob's side.
You will communicate with this server (as Alice), and implement the handshake process detailed above, over an insecure HTTP channel. 

Before you run the server on your public EC2 instance, you should install some Python packages:

```shell
sudo apt update && sudo apt install python3-pip
pip install aiohttp==3.9.3
```

The server can be run by:

```bash
python3 app.py
```

After the server is running and inbound traffic on port 8088 is allowed, you can test the server by executing the below command from your local machine:

```console
myuser@hostname:~$ curl <public-ec2-instance-ip>:8088/status
Hi! I'm available, let's start the TLS handshake
```

Your goal is to perform the above 6 steps using a bash script, and establish a secure channel with the server.

Below are some helpful instructions you may utilize in each step. Eventually, your code should be written in `tlsHandshake.sh` and executed by:

```bash
bash tlsHandshake.sh <server-ip>
```

While `<server-ip>` is the server public IP address. 

Make your script robust and clean, use variables, in every step check if the commands have succeeded and print informational messages accordingly. 

Use `curl` to send the following **Client Hello** HTTP request to the server:

```json lines
POST /clienthello
{
   "version": "1.3",
   "ciphersSuites": [
      "TLS_AES_128_GCM_SHA256",
      "TLS_CHACHA20_POLY1305_SHA256"
   ], 
   "message": "Client Hello"
}
```

`POST` is the request method, `/clienthello` is the endpoint, and the json is the body.

**Server Hello** response will be in the form:

```json
{
   "version": "1.3",
   "cipherSuite": "TLS_AES_128_GCM_SHA256", 
   "sessionID": "......",
   "serverCert": "......"
}
```

The response is in JSON format.
You may want to keep the `sessionID` in a variable, and the server cert in a file, for later usage.
Use the `jq` command to parse and save specific keys from the JSON response.

Assuming the server certificate was stored in `cert.pem` file, you can verify the certificate by:
```shell
openssl verify -CAfile cert-ca-aws.pem cert.pem
```

While `cert-ca-aws.pem` is the CA certificate file (in our case of Amazon Web Services). 
You can safely download it from: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/networking_project/cert-ca-aws.pem (`wget`...).

Upon a valid certificate validation, the following output will be printed to stdout:
```text
cert.pem: OK
```

If the verification fails, `exit` the program with exit code `5`, and print an informational message:

```text
Server Certificate is invalid.
```

Given a valid cert, generate 32 random bytes base64 string (e.g. using `openssl rand`). This string will be used as the **master-key**, save it somewhere for later usage.

Got tired? refresh yourself with some [interesting reading](https://www.bleepingcomputer.com/news/security/russia-creates-its-own-tls-certificate-authority-to-bypass-sanctions/amp/).

The bellow command can help you to encrypt the generated master-key secret with the server certificate:
```shell
openssl smime -encrypt -aes-256-cbc -in <file-contains-the-generated-master-key> -outform DER <file-contains-the-server-certificate> | base64 -w 0
```

When you are ready to send the encrypted master-key to the server, `curl` again an HTTP POST request to the server endpoint `/keyexchange`, with the following body:
```
POST /keyexchange
{
    "sessionID": SESSION_ID,
    "masterKey": MASTER_KEY,
    "sampleMessage": "Hi server, please encrypt me and send to client!"
}
```

While  `SESSION_ID` is the session ID you've got from the server's hello response.
Also, `MASTER_KEY` is your **encrypted** master key.

The response for the above request would be in the form:
```json
{
  "sessionID": ".....",
  "encryptedSampleMessage": "....."
}
```

All you have to do now is to decrypt the sample message and verify that it equals to the original sample message.
This will indicate that the server uses the master-key successfully.

Please note that the `encryptedSampleMessage` is base64 encoded, before you decrypt it, decode it using the `base64 -d` command.
Also, here is the command **used by the server** to encrypt the sample message, so you'll know which algorithm to use in order to decrypt it:

```bash
echo $SAMPLE_MESSAGE | openssl enc -e -aes-256-cbc -pbkdf2 -k $MASTER_KEY
```

You should `exit` the program upon an invalid decryption with exit code `6`, and print an informational message:

```text
Server symmetric encryption using the exchanged master-key has failed.
```

If everything is ok, print:

```text
Client-Server TLS handshake has been completed successfully
```

Well Done! you've manually implemented a secure communication over HTTP! Thanks god we have TLS in real life :-)






[networking_alice_bob]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_alice_bob.png
[networking_symmetric-key-enc]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_symmetric-key-enc.png
[networking_asymmetric-key-enc]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_asymmetric-key-enc.png
[networking_digital-signature]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_digital-signature.png
