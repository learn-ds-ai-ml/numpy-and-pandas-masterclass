# Undestanding SSH

## 1: What is SSH?

**SSH** stands for **Secure Shell**. It is a secure, encrypted network protocol used to connect to a remote computer (like a server or a GitHub repository) over the internet.

Think of SSH as a secure, invisible tunnel between your local computer and a remote machine. Everything you send through this tunnel is encrypted, so hackers cannot steal your passwords or code.

---

## 2: SSH Keys

### 2.1: Whate are SSH Keys?

Instead of using a traditional username and password, SSH uses **SSH Keys**. SSH keys come in matching pairs called a **Key Pair**.

To understand how they work, use the analogy of a **Lock and a Key**:

| **Component**                     | **Analogy**          | **What it does**                                                                                                                                         |
| --------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **🔓 Public Key** (`.pub` file)   | **The Lock**         | You give this to the world. You paste it onto GitHub or a remote server. Anyone can see it. It acts like a padlock that only your specific key can open. |
| **🔑 Private Key** (no extension) | **The Physical Key** | You keep this hidden on your local computer. **Never share it with anyone**. It is the only key that can open your Public Padlock.                       |

---

### 2.2: How to Create a New SSH Key

To create a key, you use the `ssh-keygen` command on your **host terminal**.

1. Run this command, replacing the email with your GitHub email:

   ```bash
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

   _(The `-t` flag in the command stands for **Type**. By running `-t ed25519`, you are telling your computer: "I want to create a key using the Ed25519 formula, rather than older formulas like RSA. [Read More](#6-why-ed25519-algorithm))_

2. **Crucial Step for Multiple Keys:** The terminal will ask where to save the file:

   ```text
   Enter file in which to save the key (/home/user/.ssh/id_ed25519):
   ```

   Do **not** just press Enter (which overwrites your default key). Instead, type a unique path and name, like:

   ```text
   /home/user/.ssh/id_github_personal
   ```

   _(For your second key, you might name it `/home/user/.ssh/id_github_work`)._

3. Enter a passphrase (or press Enter twice to leave it blank).

---

### 2.3: Managing Multiple Accounts via the SSH Config File

If you try to connect to GitHub normally, your computer will just send your default key. To fix this, you create an SSH configuration file that acts like a traffic controller.

1. Create or open the config file on your host machine:

   ```bash
   nano ~/.ssh/config
   ```

2. Paste the following configuration structure. This creates custom "nicknames" (Hosts) for GitHub:

   ```text
   # --- Personal GitHub Account ---
   Host github.com-personal
     HostName github.com
     User git
     IdentityFile ~/.ssh/id_github_personal
     IdentitiesOnly yes

   # --- Work GitHub Account ---
   Host github.com-work
     HostName github.com
     User git
     IdentityFile ~/.ssh/id_github_work
     IdentitiesOnly yes
   ```

3. Save and close the file (in Nano, press `Ctrl+O`, `Enter`, then `Ctrl+X`).

---

### 2.4: How to Clone and Use the Repositories

Because you set up custom Host nicknames (`github.com-personal` and `github.com-work`), you must slightly alter the URL when you clone a repository.

#### 2.4.1: When cloning a Personal repo

Instead of copying the exact standard link from GitHub (`git@github.com:username/repo.git`), change `github.com` to your personal nickname `github.com-personal`:

```text
git clone git@github.com-personal:personal_username/my-cool-project.git
```

---

## 2.4.2: When cloning a Work repo

Change `github.com` to your work nickname:

```text
git clone git@github.com-work:work_username/company-project.git
```

---

Your computer reads that nickname, opens the `~/.ssh/config` file, translates it back to the real `github.com`, and forces the correct private key to handle the login.

---

### 2.5: Keep Your Git Commits Separate (Bonus Tip)

Even if your SSH keys are perfectly separated, Git might still stamp your commits with a single global email address. To prevent your personal email from showing up on work commits (or vice versa), set local configs inside each repository folder.

Once you clone a repository, open its terminal and run:

```bash
# Inside a work project folder
git config user.name "Your Name"
git config user.email "your_work_email@company.com"

# Inside a personal project folder
git config user.name "Your Name"
git config user.email "your_personal_email@gmail.com"
```

_(Removing the `--global` flag ensures the setting applies only to that exact project folder)._

---

## 3: How SSH Authentication Works (The Handshake)

When you run a command like `git push` or `ssh -T git@github.com`, your computer and GitHub perform a quick automatic "handshake" using advanced mathematics (Asymmetric Cryptography).

Here is exactly what happens behind the scenes in a matter of milliseconds:

```text
 [ Your Computer ]                                       [ GitHub ]
  (Private Key)                                        (Public Key)
        |                                                     |
        | ------------ 1. "Hey, let me in!" ----------------> |
        |                                                     |
        | <--- 2. Sends an encrypted random puzzle ---------  | (Locked with your Public Key)
        |                                                     |
        | 3. Unlocks puzzle using Private Key                 |
        | 4. Solves it & signs it                             |
        |                                                     |
        | ------------ 5. Sends the solution back ----------> |
        |                                                     |
        |                                             6. Verifies solution.

        | <----------- 7. "Access Granted!" ----------------- |
```

1. **The Request:** Your computer contacts GitHub and says, _"Hey, I want to log in as user 'john_doe'."_
2. **The Challenge:** GitHub looks up the **Public Key** you pasted into your account settings. GitHub creates a random mathematical puzzle, locks it with that Public Key, and sends it back to your computer.
3. **The Decryption:** Your computer receives the locked puzzle. Because your computer holds the matching Private Key, it is able to easily unlock the puzzle, solve it, and sign it with a digital signature.
4. **The Verification:** Your computer sends the solution back to GitHub. GitHub verifies the solution using your public key. If the math matches perfectly, GitHub knows for a fact that you own the private key.
5. **Access Granted:** GitHub lets you pull or push code without you ever having to type a password!

---

## 4: Why Use SSH Keys Instead of Passwords?

- **Uncrackable:** Passwords can be guessed, phished, or cracked by brute-force computers. SSH keys are incredibly complex, massive mathematical numbers that are virtually impossible to guess.
- **Automation Friendly:** Because the handshake happens automatically between your computer and GitHub, scripts and developer tools (like Docker and Jupyter) can interact with your code seamlessly without stopping to ask you for your login password every time.

---

## 5: SSH-Client & SSH-Agent

**ssh-client and ssh-agent are two completely different programs that work together as a team**.

The easiest way to understand the difference is using a workplace analogy:

- **The SSH-Client is the Worker:** It travels, establishes the connection, and does the heavy lifting.
- **The SSH-Agent is the Wallet/Key Guard:** It safely holds onto your keys, unlocks them for the worker, and keeps them safe in memory.

Here is the detailed breakdown of their specific roles:

---

### 5.1: The SSH-Client (The Program)

The `ssh-client` is the actual core tool you use when you type commands like `ssh`, `scp`, or `git push` (since Git uses the SSH-client under the hood).

- **What it does:** It creates the encrypted tunnel, talks to GitHub, receives the mathematical challenge puzzle, and sends back the answer.
- **Its limitation:** It is forgetful and lazy. Every single time it needs to sign a challenge, it has to manually go to your hard drive, read your private key file, and—if the key is encrypted—stop everything to ask you for your passphrase.

---

### 5.2: The SSH-Agent (The Helper)

The `ssh-agent` is a background helper program (a "daemon") that runs silently in your computer's temporary memory (RAM).

- **What it does:** It holds your unlocked private keys in memory so you don't have to keep reading them from the disk.
- **How it works with the client:** When you run `ssh-add ~/.ssh/id_ed25519`, you type your password **once**. The agent unlocks the key and holds onto it. Later, when the `ssh-client` needs to talk to GitHub, it simply asks the agent: _"Hey, can you sign this puzzle for me using the GitHub key?"_ The agent does the math in memory and hands back the signature.

---

### 5.3: Summary Comparison

| **Feature**             | **ssh-client (The Command)**                              | **ssh-agent (The Key Guard)**                                 |
| ----------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| **Type of Program**     | An active command-line tool.                              | A passive background process.                                 |
| **Core Job**            | Connects to remote servers and runs commands.             | Safely stores and manages unlocked keys in RAM.               |
| **Passphrase Handling** | Asks you for your password every single time you connect. | Asks you for your password only once when you add the key.    |
| **Awareness**           | Knows where you want to connect (e.g., github.com).       | Doesn't care where you are connecting; only manages the keys. |

---

### 5.4: How they talk to each other (The Socket)

When you start the agent using eval `"$(ssh-agent -s)"`, it creates a temporary communications channel called a **Unix Socket** on your computer. It saves the path to this socket in an environment variable called `SSH_AUTH_SOCK`.

Whenever the `ssh-client` runs, it automatically looks for the `SSH_AUTH_SOCK` variable. If it finds it, it routes all key requests through the agent.

_This is exactly why we passed SSH_AUTH_SOCK into your Docker container earlier! It allowed the container's ssh-client to reach out across the socket and talk directly to your host's ssh-agent._

---

## 6: why `ed25519` algorithm?

`ed25519` is the name of the **mathematical algorithm** used to generate your SSH key pair.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

The `-t` flag in the command stands for **Type**. By running `-t ed25519`, you are telling your computer: _"I want to create a key using the Ed25519 formula, rather than older formulas like RSA."_

Today, `ed25519` is universally considered the **gold standard** for SSH keys by cybersecurity experts. Here is why it is so special, explained simply:

---

### 6.1: It is Incredibly Small and Fast

An Ed25519 key is visually much shorter than an older RSA key.

- A standard **RSA key** looks like a massive wall of text (typically 3,072 or 4,096 characters long).
- An **Ed25519 key** is a tiny, neat string of just 68 characters.

Because it is so small, your computer can process the authentication math **much faster**, using fewer CPU cycles. This is highly efficient for automated systems and Docker containers.

### 6.2: It offers Top-Tier Security

You might think a smaller key is easier to crack, but Ed25519 uses **Elliptic Curve Cryptography (ECC)** instead of factoring giant prime numbers (which RSA uses).

A tiny **256-bit Ed25519 key** provides the exact same level of cryptographic security as a massive **3072-bit RSA key**. It is mathematically impossible for modern supercomputers to brute-force or guess.

### 6.3: It is Immune to Side-Channel Attacks

Ed25519 was specifically designed by a famous cryptographer named Daniel J. Bernstein to be safe against "side-channel attacks."

In complex computing, hackers can sometimes figure out a secret key by measuring the exact fractions of a millisecond a processor takes to do math, or by looking at power consumption changes. Ed25519 performs its math in **constant time**—it takes the exact same number of CPU cycles every time, leaving no clues for hackers to exploit.

---

### 6.4: Summary of Key Types (Quick Comparison)

| **Key Type**  | **Security Status**        | **Verdict**                                                                                                                                |
| ------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **`ed25519`** | **🏆 Maximum Security**    | The best choice. Fastest, smallest, and highly secure.                                                                                     |
| **`rsa`**     | **⚠️ Legacy / Deprecated** | Secure only if configured to 3072 bits or higher. GitHub has dropped support for older, smaller RSA keys entirely because they are unsafe. |
| **`ecdsa`**   | **🛑 Not Recommended**     | An older elliptic curve standard created by the US Government (NIST). Some cryptographers worry it contains hidden backdoors.              |
