# Fixing SSH Access to Gitea: A Complete Walkthrough

## The Problem

I was unable to push or pull code from my Gitea instance running on a local server (Battlemage, IP `192.168.18.51`) using the SSH protocol. Every attempt to authenticate via SSH with `git@192.168.18.51` resulted in the dreaded `Permission denied (publickey,password)` error, even though my public key had already been added to the Gitea web interface under my user account (username: `dex`). The initial investigation revealed that the Git remote was configured as `git@gitea.exclusion.cc:dex/postgreSQL-learning.git`, which through Tailscale DNS resolution pointed to the same server at `192.168.18.51`. Despite the DNS being correct and the key being registered, SSH authentication consistently failed, leaving me locked out of my repositories and unable to collaborate on my PostgreSQL learning project through the terminal.

## Initial Diagnostics

The first step was to methodically examine each layer of the SSH connection to isolate precisely where the failure was occurring. I began by conducting a verbose SSH connection test targeting port 22, the standard SSH port, on the Gitea server at `192.168.18.51`. The verbose output from OpenSSH version 10.2p1 revealed several crucial pieces of information. First, the connection to the server was established successfully at the TCP level, confirming that network connectivity was not the issue and that the Tailscale VPN routing was functioning correctly. Second, my local SSH client correctly loaded my RSA key pair from `~/.ssh/id_rsa` and offered it to the server as an authentication method. The server's remote software banner identified itself as `OpenSSH_9.6p1 Ubuntu-3ubuntu13.16`, which immediately raised a red flag because I knew Gitea was running inside a Docker container, not directly on the host operating system. The SSH protocol handshake completed without issues, including key exchange using the `sntrup761x25519-sha512@openssh.com` algorithm and host key verification against the known hosts file. However, the critical debug line `debug1: Offering public key: /c/Users/Yaku/.ssh/id_rsa RSA SHA256:FB/gFGKx5nUGLlRM7JBGE5KSTMybI/GZJfdoWiy5Qu0` was immediately followed by the server rejecting the key with `debug1: Authentications that can continue: publickey,password`. This pattern repeated for all available keys, and the SSH client eventually exhausted its authentication methods and disconnected.

This behavior was puzzling because the Gitea API confirmed that the exact same public key was indeed registered in Gitea's database for user `dex`. When I attempted to add the key again through the API endpoint `POST /api/v1/user/keys`, the server responded with the message `"Key content has been used as non-deploy key"`, which is Gitea's way of saying the key already exists in the system. This contradiction — the key being present in Gitea's database but rejected during SSH authentication — was the core mystery that needed to be solved.

## Investigating the Server Architecture

To understand why SSH authentication was failing despite the key being registered, I needed to examine the server-side configuration directly. Using a Python-based SSH connection to the Battlemage server (authenticating with standard password credentials rather than key-based authentication), I conducted a thorough reconnaissance of the Docker infrastructure.

Executing `docker ps` on the server revealed a sprawling ecosystem of over forty Docker containers running various services including media servers, wikis, note-taking applications, monitoring tools, and importantly, the Gitea container itself. The Gitea container was named `gitea` and was running the image `docker.gitea.com/gitea:1.24.5`. Critically, the output also displayed the port mappings for every container, and the Gitea container's port mapping was listed as `222:22` — meaning that port 222 on the Docker host was mapped to port 22 inside the Gitea container. This was the smoking gun. When I was connecting to port 22 on the host IP address, I was not reaching Gitea's internal SSH server but rather the host machine's own OpenSSH daemon running on Ubuntu. The host's SSH server had its own set of authorized keys for the `git` user, and my key was not among them, which explained the persistent authentication failures.

I then located the Docker Compose configuration file at `/home/yaku/gitea/docker-compose.yml`. The file contents confirmed the port mapping:

```yaml
networks:
  gitea:
    external: false

services:
  server:
    image: docker.gitea.com/gitea:1.24.5
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3003:3003"
      - "222:22"
```

This configuration explicitly shows that the web interface is exposed on host port 3003 (mapped to container port 3003), and the SSH service is exposed on host port 222 (mapped to container port 22). The Docker internal SSH server inside the Gitea container listens on the standard port 22 within the container's network namespace, but from the perspective of the host machine and the external network, it is accessible exclusively through port 222. Any SSH connection attempting port 22 on the host reaches the Ubuntu host's native OpenSSH server, which has no knowledge of the keys stored in Gitea's database.

## The Configuration Mismatch

The next step was to examine Gitea's own application configuration file to understand why it was generating incorrect SSH URLs. I retrieved the `app.ini` file from within the running Gitea container at `/data/gitea/conf/app.ini`. The relevant section of the configuration read:

```ini
[server]
APP_DATA_PATH  = /data/gitea
DOMAIN         = 192.168.18.51
SSH_DOMAIN     = 192.168.18.51
HTTP_PORT      = 3003
ROOT_URL       = http://192.168.18.51:3003/
DISABLE_SSH    = false
SSH_PORT       = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
```

The critical parameter here is `SSH_PORT = 22`. In Gitea's configuration, `SSH_PORT` serves a dual purpose: it specifies the port on which the internal SSH server listens (which, combined with `SSH_LISTEN_PORT = 22`, tells the embedded SSH daemon to bind to port 22 inside the container), but more importantly for this issue, it determines the port number that Gitea displays in SSH clone URLs throughout the web interface and the API. Because this value was set to `22`, Gitea was telling users — and itself — that the SSH service was available on the standard port 22 of the host, when in reality, due to the Docker port mapping, the service was only reachable on port 222 from outside the container.

This created a specific failure scenario. The web interface at `http://192.168.18.51:3003` would display SSH clone URLs like `ssh://git@192.168.18.51:22/dex/postgreSQL-learning.git`. When a user copied this URL and attempted to clone or set it as a remote, their SSH client would connect to port 22 on `192.168.18.51`, where the Ubuntu host's OpenSSH server was listening. The host's SSH server, not having the user's public key in its `authorized_keys` file for the `git` user, would reject the connection with a permission denied error. The user, having already added their key through the Gitea web interface, would be left confused as to why authentication was failing when their key was clearly registered in the system.

The `SSH_LISTEN_PORT = 22` parameter is separate and correctly set because it controls the internal listening port inside the container, which should indeed be 22 since the Docker mapping `222:22` translates the external port 222 to internal port 22. The fix therefore only needed to target the `SSH_PORT` parameter, which controls the externally visible port number used in generated URLs.

## The Fix

The solution involved two coordinated changes: one on the server to correct the Gitea configuration, and one on the client to update the Git remote URL.

### Server-Side Fix

On the Battlemage server, I modified the Gitea application configuration file located at `/home/yaku/gitea/gitea/gitea/conf/app.ini` (mounted as a Docker volume from the host into the container's `/data/gitea/conf/app.ini`). Using the `sed` stream editor, I changed the `SSH_PORT` parameter from `22` to `222`:

```bash
sed -i "s/SSH_PORT = 22/SSH_PORT = 222/" /home/yaku/gitea/gitea/gitea/conf/app.ini
```

This change tells Gitea that its SSH service is accessible on port 222 from the outside world. The web interface and API will now generate correct SSH clone URLs such as `ssh://git@192.168.18.51:222/dex/postgreSQL-learning.git`. Note that the `SSH_LISTEN_PORT` parameter was left unchanged at 22 because the internal SSH daemon still needs to bind to port 22 inside the container — the Docker networking layer handles the translation between the external port 222 and the internal port 22.

After modifying the configuration file, I restarted the Gitea container to apply the changes:

```bash
cd /home/yaku/gitea && docker compose restart server
```

This triggers a graceful restart of the Gitea container, during which it reloads its configuration from the updated `app.ini` file.

### Client-Side Fix

On the local Windows development machine, I updated the Git remote URL for the repository to use the correct SSH port. The previous remote URL had been pointing to the standard SSH port implicitly, which meant it was connecting to port 22 (the host's SSH server). The new remote URL explicitly specifies port 222 using the SSH URL scheme:

```bash
git remote set-url origin ssh://git@192.168.18.51:222/dex/postgreSQL-learning.git
```

I also needed to add the host key for port 222 to the SSH known hosts file, since the SSH service on port 222 presents a different host key than the Ubuntu host's SSH on port 22 (it is Gitea's embedded SSH server generating its own host key):

```bash
ssh-keyscan -p 222 192.168.18.51 >> ~/.ssh/known_hosts
```

## Verification

After applying both fixes, I performed a comprehensive verification to ensure that SSH authentication and Git operations were functioning correctly.

First, I tested the raw SSH connection using the correct port:

```
$ ssh -T -p 222 git@192.168.18.51
Hi there, dex! You've successfully authenticated with the key named Windows 11,
but Gitea does not provide shell access.
```

This is the expected response from a Gitea SSH server when authentication succeeds. The message confirms three things: the SSH connection was established successfully, the public key was properly matched against Gitea's internal database of registered keys for user `dex`, and Gitea correctly identified the connecting user. The note about shell access being unavailable is normal behavior — Gitea does not provide interactive shell sessions; it only handles Git operations through SSH.

Second, I verified the Git remote configuration:

```
$ git remote -v
origin  ssh://git@192.168.18.51:222/dex/postgreSQL-learning.git (fetch)
origin  ssh://git@192.168.18.51:222/dex/postgreSQL-learning.git (push)
```

The remote URL now correctly specifies port 222 using the SSH protocol scheme.

Finally, I tested the complete Git remote communication using `git ls-remote origin`, which successfully listed references from the remote repository, confirming that all Git operations — fetch, pull, and push — would work correctly through the SSH connection on port 222.

## Root Cause Summary

The root cause of the SSH authentication failure was a mismatch between the port that Docker exposed for Gitea's SSH service (port 222 on the host) and the port that Gitea was configured to advertise in its SSH URLs (port 22 in `app.ini`). This mismatch caused all SSH-based Git operations to target the Ubuntu host's native OpenSSH server on port 22 instead of the Gitea container's embedded SSH server on port 222. The host's SSH server had no access to Gitea's internal user database and authorized keys, leading to persistent authentication failures despite the user's public key being registered correctly through the Gitea web interface. The fix was a simple configuration change — updating `SSH_PORT` from `22` to `222` in Gitea's `app.ini` to reflect the actual Docker port mapping — followed by restarting the Gitea container and updating the client-side Git remote URL to use the correct port.

## Lessons Learned

This troubleshooting exercise illustrates several important principles for diagnosing SSH connectivity issues in Docker-based infrastructure. First, when a service runs inside a container with port mappings, the port visible to external clients is the host port, not the container's internal port. The Docker port mapping directionality (host:container) must be understood correctly — `222:22` means host port 222 maps to container port 22, not the reverse. Second, the `SSH_PORT` configuration parameter in Gitea serves a different purpose from `SSH_LISTEN_PORT`: the former controls what appears in generated URLs and should reflect the externally accessible port, while the latter controls the actual binding port inside the container. Third, verbose SSH debugging (`ssh -vvv`) combined with API-level verification (checking key registration status through the Gitea API) can quickly isolate whether an authentication failure is caused by network issues, key registration problems, or a port mismatch. Finally, when dealing with Docker-hosted services, always verify the container's port mappings with `docker ps` or by reading the `docker-compose.yml` file before debugging authentication issues, as misrouted connections to the wrong service are a common and easily overlooked source of failures.
