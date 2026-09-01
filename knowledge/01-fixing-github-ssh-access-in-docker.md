# 📋 Cheat Sheet: Fixing GitHub SSH Access in Docker

## The Problem

The Jupyter container runs as user `jovyan` (UID `1000`) and cannot read your host’s restricted `~/.ssh` folder, or it repeatedly asks you for an unknown key passphrase.

---

## 🔍 Step 1: Find the Right SSH Key on Your Host

If you have multiple keys, run this on your **host terminal** to find which one is linked to GitHub:

```bash
# You will see a list of files. Look for pairs of files with names like these:
# - Modern key type: `id_ed25519` (private key) and `id_ed25519.pub` (public key)
# - Older key type: `id_rsa` (private key) and `id_rsa.pub` (public key)
# The file without `.pub` at the end is your private key name (e.g., `id_ed25519` or `id_rsa`). That is the name you use when setting up the SSH agent.
ls -la ~/.ssh

# Test a specific key (replace 'id_ed25519' with your key file name)
ssh -i ~/.ssh/id_ed25519 -T git@github.com
```

- **Success:** Shows your GitHub username.
- **Failure:** `Says Permission denied (publickey)`.

_Alternatively, match the fingerprint output of `for key in ~/.ssh/*.pub; do ssh-keygen -lf "$key"; done` with the keys listed in your **GitHub Settings -> SSH and GPG keys**._

## 🛠️ Step 2: The Best Fix (SSH Agent Forwarding)

Instead of copying files or changing host permissions, let your host machine handle the key unlocking and securely "share" the connection with Docker.

### 1. Update `docker-compose.yml`

```yml
services:
  jupyter-scipy:
    # Jupyter's SciPy notebook image.
    # This image already contains Python, Jupyter,
    # SciPy, NumPy, pandas, matplotlib, etc.
    image: quay.io/jupyter/scipy-notebook:latest
    environment:
      # ------------------------------------------------------------------
      # 6. Tell SSH where the agent socket is INSIDE the container
      # ------------------------------------------------------------------
      # SSH_AUTH_SOCK points to a Unix socket that SSH uses to communicate with an SSH agent.
      # When you run ssh, git, or another SSH-enabled program,
      # it can use that socket to ask the agent to authenticate without directly accessing your private key.
      #
      # ------------
      #  ssh
      #   ↓
      #  SSH_AUTH_SOCK
      #   ↓
      #  ssh-agent
      #   ↓
      #  private key
      #------------
      #
      # On the HOST, SSH_AUTH_SOCK might point to something like:
      #
      #   /run/user/1000/keyring/ssh
      #
      # That path generally does NOT exist inside the container.
      #
      # In the Volumes section below, we mount the host's SSH agent socket into the container at:
      #
      #   /run/host-ssh-agent
      #
      # inside the container.
      #
      # Setting SSH_AUTH_SOCK tells SSH:
      #
      #   "Use this socket when communicating with my SSH agent."
      #
      # So the complete flow is:
      #
      #   SSH inside container
      #          |
      #          v
      #   SSH_AUTH_SOCK=/run/host-ssh-agent
      #          |
      #          v
      #   Docker-mounted host agent socket
      #          |
      #          v
      #   SSH agent on host
      #          |
      #          v
      #   Private key on host
      #
      - SSH_AUTH_SOCK=/run/host-ssh-agent

    ports:
      # HOST:CONTAINER
      #
      # Jupyter listens on port 8888 INSIDE the container.
      # We expose it as port 8889 on the host.
      #
      # Open Jupyter at:
      #   http://localhost:8889
      - "8889:8888"

    volumes:
      # ------------------------------------------------------------------
      # 1. Mount the current project directory
      # ------------------------------------------------------------------
      #
      # "." means the directory containing this docker-compose.yml file.
      #
      # "/home/jovyan/work" is the directory inside the Jupyter container
      # where the Jupyter image expects user/project files.
      #
      # This is a BIND MOUNT, so changes are immediately visible on both
      # sides:
      #
      #   HOST                         CONTAINER
      #   ./                           /home/jovyan/work
      #
      # Files created/modified in the container are therefore also present
      # on the host.
      #
      - .:/home/jovyan/work

      # ------------------------------------------------------------------
      # 2. Mount the host's Git configuration
      # ------------------------------------------------------------------
      #
      # ~/.gitconfig contains your Git configuration, for example:
      #
      #   [user]
      #       name = Your Name
      #       email = you@example.com
      #
      #   [core]
      #       editor = ...
      #
      # It is mounted as /home/jovyan/.gitconfig because "jovyan" is the
      # user running Jupyter inside this image.
      #
      # ":ro" means READ-ONLY.
      #
      # Therefore:
      #   Container can READ the host's .gitconfig
      #   Container cannot MODIFY the host's .gitconfig through this mount
      #
      - ~/.gitconfig:/home/jovyan/.gitconfig:ro

      # ------------------------------------------------------------------
      # 3. Mount the HOST SSH AGENT socket
      # ------------------------------------------------------------------
      #
      # SSH_AUTH_SOCK on the host points to a Unix socket belonging to
      # your SSH agent.
      #
      # IMPORTANT:
      # We are NOT mounting ~/.ssh/id_ed25519 or any private SSH key.
      #
      # Instead, we expose the SSH agent itself to the container.
      #
      # Conceptually:
      #
      #   Container
      #       |
      #       | SSH request
      #       v
      #   /run/host-ssh-agent
      #       |
      #       | Docker bind mount
      #       v
      #   Host SSH agent socket
      #       |
      #       v
      #   Private key stays on HOST
      #
      # This allows Git/SSH inside the container to authenticate using
      # keys loaded into your host's SSH agent without copying the private
      # key into the container.
      #
      # ":ro" prevents the container from modifying the mounted socket
      # through the filesystem mount. The socket still provides the
      # communication channel to the SSH agent.
      #
      - ${SSH_AUTH_SOCK}:/run/host-ssh-agent:ro

      # ------------------------------------------------------------------
      # 4. Mount SSH configuration
      # ------------------------------------------------------------------
      #
      # ~/.ssh/config contains SSH client configuration from the host.
      #
      # It can contain things such as:
      #
      #   Host github.com
      #       User git
      #       IdentityFile ...
      #
      # or aliases, ports, ProxyJump settings, etc.
      #
      # We mount it into the location where SSH running as "jovyan"
      # expects its configuration:
      #
      #   HOST:      ~/.ssh/config
      #   CONTAINER: /home/jovyan/.ssh/config
      #
      # ":ro" means the container can read it but cannot modify the
      # host's SSH configuration.
      #
      - ~/.ssh/config:/home/jovyan/.ssh/config:ro

      # ------------------------------------------------------------------
      # 5. Mount known_hosts
      # ------------------------------------------------------------------
      #
      # known_hosts contains SSH host keys that your host has previously
      # trusted.
      #
      # It allows SSH inside the container to verify that a server such as
      # GitHub is the server it is supposed to be connecting to.
      #
      # Without this, SSH may stop with:
      #
      #   Host key verification failed
      #
      # Again, ":ro" means the container can read this file but cannot
      # modify the host's copy.
      #
      - ~/.ssh/known_hosts:/home/jovyan/.ssh/known_hosts:ro
```

### 2. Start the Agent & Run (On Host Terminal)

```bash
# Stop old container
docker compose down

# Start host SSH agent and add your verified key (unlocks it once)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/YOUR_KEY_NAME # e.g., id_ed25519 or id_rsa

# Boot up container
docker compose up
```

#### 2.1: How to check how many keys agent is holding?

```bash
ssh-add -l
```

---

## 🆕 Alternative: Generate a Fresh Key Inside Jupyter

If you completely forgot your host key's passphrase, just make a brand new, passwordless key inside the container.

1. Open **Terminal** inside JupyterLab (**File -> New -> Terminal**).
2. Run the generator (press **Enter** for all prompts to leave the passphrase blank):

   ```bash
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

3. Print and copy the new public key:

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

4. Paste this full string into **GitHub -> Settings -> SSH and GPG Keys -> New SSH Key**.

---

## 🧪 Step 3: Verify the Connection

Inside your **Jupyter Lab Terminal**, test that everything works by running:

```bash
ssh -T git@github.com
```

**🎯 Expected Success Message:** _"Hi username! You've successfully authenticated, but GitHub does not provide shell access."_
