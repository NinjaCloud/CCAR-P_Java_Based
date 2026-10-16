# Lab 0: Set Up Your Laptop (Java, Maven, Claude Code, API Key) - 20 minutes

Do this lab **before** the multi-agent labs. It works on **Windows (PowerShell)** and **macOS (Terminal, zsh)**. Follow only the tab for your computer; steps that are identical on both are shown once.

By the end you will have:

- JDK 21 and Maven 3.9+ working
- Claude Code installed and logged in with your API key
- A passing end-to-end check

| Step | What | Time |
|------|------|------|
| 1 | Get your API key | 3 min |
| 2 | Install Java (JDK 21) | 5 min |
| 3 | Install Maven | 4 min |
| 4 | Install Claude Code | 3 min |
| 5 | Configure the API key | 3 min |
| 6 | Final verification | 2 min |

> **Security rules for the API key**
> - It is a password. Never paste it into code, a README, a screenshot, a chat message or Git.
> - If it leaks, revoke it in the Claude Console and create a new one.
> - On shared or classroom machines use the session-only setting in Step 5A, not the permanent one.

---

## Step 1 - Get your API key (3 min)

1. Open https://platform.claude.com and sign in to the Claude Console (your trainer may have given you an invite or a key; if so, skip to Step 2).
2. Go to **Settings -> API keys** and click **Create key**.
3. Name it `claude-training` and copy the key now. It starts with `sk-ant-` and is shown only once.
4. Keep it in a password manager or a temporary note, not in a file inside a project folder.

---

## Step 2 - Install Java (JDK 21) (5 min)

### Windows (PowerShell)

Open **PowerShell** (Start menu -> type "PowerShell"). Administrator is not required.

```powershell
winget install Microsoft.OpenJDK.21
```

Close PowerShell and open a **new** window so the PATH refreshes, then check:

```powershell
java -version
```

Expected: a line starting with `openjdk version "21...`.

### macOS (Terminal)

Open **Terminal**. First check whether Homebrew (the macOS package manager) is installed:

```bash
brew --version
```

If you get `command not found`, install it (follow the instructions it prints at the end, which add Homebrew to your PATH):

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install the JDK:

```bash
brew install --cask temurin@21
```

Tell your shell where Java lives, then reload the shell config:

```bash
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 21)' >> ~/.zshrc
source ~/.zshrc
java -version
```

Expected: a line starting with `openjdk version "21...`.

**What this step did:** it installed the Java Development Kit, which compiles and runs the lab code. `JAVA_HOME` is the variable Maven uses to find Java.

---

## Step 3 - Install Maven (4 min)

Maven builds the Java projects and downloads the Claude SDK.

### macOS (Terminal)

```bash
brew install maven
mvn -v
```

Check that the line `Java version:` shows **21**. If it shows a different version, run `source ~/.zshrc` and try again.

### Windows (PowerShell)

1. In your browser open https://maven.apache.org/download.cgi and download the **Binary zip archive** (`apache-maven-3.9.x-bin.zip`) into your `Downloads` folder.
2. Unzip it to `C:\tools` and register Maven and Java for your user account:

```powershell
New-Item -ItemType Directory -Force C:\tools | Out-Null
Expand-Archive "$env:USERPROFILE\Downloads\apache-maven-*-bin.zip" -DestinationPath C:\tools -Force

$mvn = (Get-ChildItem C:\tools -Directory -Filter "apache-maven-*" | Select-Object -First 1).FullName
$jdk = (Get-ChildItem "C:\Program Files\Microsoft" -Directory -Filter "jdk-21*" | Select-Object -First 1).FullName

[Environment]::SetEnvironmentVariable("JAVA_HOME", $jdk, "User")
[Environment]::SetEnvironmentVariable("MAVEN_HOME", $mvn, "User")
$path = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path", "$path;$mvn\bin", "User")
```

3. Close PowerShell, open a **new** window and check:

```powershell
mvn -v
```

Expected: `Apache Maven 3.9...` and `Java version: 21...`.

**What this step did:** it unpacked Maven, then stored `JAVA_HOME`, `MAVEN_HOME` and the PATH entry so every new terminal can run `mvn`.

---

## Step 4 - Install Claude Code (3 min)

Claude Code is Anthropic's coding assistant for the terminal. It does not need Node.js or Java.

### Windows (PowerShell)

```powershell
irm https://claude.ai/install.ps1 | iex
```

(Optional but recommended: `winget install Git.Git`. Claude Code uses Git Bash if it is present. Without it, Claude Code uses PowerShell.)

### macOS (Terminal)

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### Both systems

The installer prints no progress while it downloads. When it finishes, **open a new terminal window** and run:

```
claude --version
```

Expected: a version number such as `2.1.x (Claude Code)`.

If you see `command not found` or `claude is not recognized`, the install folder is not on your PATH yet. Close and reopen the terminal first. If it still fails, run the install command again and read its last lines.

Run a health check:

```
claude doctor
```

**Alternative installs** (use only if your trainer says so): macOS `brew install --cask claude-code`, Windows `winget install Anthropic.ClaudeCode`.

---

## Step 5 - Configure the API key (3 min)

Both the Claude Code CLI and the Java SDK read the same environment variable: **`ANTHROPIC_API_KEY`**.

### 5A. Session-only (recommended for classrooms and shared machines)

The key exists only in the terminal window where you set it. Closing the window removes it.

**Windows PowerShell**

```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-your-key-here"
```

**macOS (zsh)**

```bash
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
```

You must repeat this in every new terminal window before running the labs.

### 5B. Permanent (only on your own personal laptop)

**Windows PowerShell** (stored for your user account, visible in *new* windows only)

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-your-key-here", "User")
```

Open a new PowerShell window afterwards.

**macOS (zsh)**

```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-your-key-here"' >> ~/.zshrc
source ~/.zshrc
```

To remove it later: delete that line from `~/.zshrc` (macOS) or run `[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "User")` (Windows).

### 5C. First start of Claude Code

Go to any folder (for example your Desktop) and run:

```
claude
```

Claude Code notices `ANTHROPIC_API_KEY` and asks whether to use it. Choose **Yes** and your choice is remembered. You skip the browser login.

Inside Claude Code, check which login is active:

```
/status
```

It should show that the API key is the method in use. Leave the session with `/exit`.

Useful commands:

| Command | Purpose |
|---------|---------|
| `/status` | Show which credential and account are active |
| `/config` | Settings; the "Use custom API key" toggle appears while `ANTHROPIC_API_KEY` is set |
| `/logout` | Sign out and start the login flow again next time |

**If you also have a Claude subscription:** once you approve the key, Claude Code uses the key (and bills the API account). To go back to your subscription, remove the variable (`Remove-Item Env:ANTHROPIC_API_KEY` in PowerShell, `unset ANTHROPIC_API_KEY` in bash) and start `claude` again.

---

## Step 6 - Final verification (2 min)

Run these in a **new** terminal with the key set (Step 5A or 5B). The same commands work on both systems.

```
java -version
mvn -v
claude --version
claude -p "Reply with the single word OK"
```

Expected:

| Command | You should see |
|---------|----------------|
| `java -version` | `openjdk version "21...` |
| `mvn -v` | `Apache Maven 3.9...` and `Java version: 21...` |
| `claude --version` | a version number |
| `claude -p "..."` | `OK` (a real reply from Claude, which proves the key works) |

**You are ready for Lab 1 when all four commands pass.**

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `java`, `mvn` or `claude` not recognized / not found | Close the terminal and open a new one. On Windows, re-run the Step 3 PowerShell block if `mvn` is still missing. |
| `mvn -v` says `JAVA_HOME` is not defined or wrong | Windows: re-run the `JAVA_HOME` line in Step 3 and open a new window. macOS: `source ~/.zshrc`. |
| `mvn -v` shows Java 17 or 8, not 21 | Your `JAVA_HOME` points to an older JDK. Re-run the `JAVA_HOME` command for your system. |
| `401`, `invalid x-api-key` or `authentication_error` | The key is missing, mistyped or revoked. Set it again in this terminal (Step 5A) and make sure there are no spaces or quotes inside the key. |
| Claude Code opens a browser login instead of using the key | The variable is not set in this terminal. Set it first, then run `claude`. |
| `claude -p` works but the key is rejected in Java later | Check the same terminal window has `ANTHROPIC_API_KEY` set: PowerShell `echo $env:ANTHROPIC_API_KEY`, bash `echo $ANTHROPIC_API_KEY` (do not share a screenshot of this). |
| PowerShell: `irm` is not recognized | You are in CMD. Open **PowerShell** instead. |
| Download or install blocked on a company network | Ask your trainer for the offline installers or proxy settings. |

---

## Key takeaways

1. The labs need three things on your machine: JDK 21, Maven, and the `ANTHROPIC_API_KEY` variable.
2. The Java SDK and Claude Code share the same variable, so one key setup serves both.
3. Prefer session-only keys on shared machines and never commit keys to Git.
4. Verify with four commands before every class: `java -version`, `mvn -v`, `claude --version`, `claude -p "..."`.

## Sources

- Claude Code setup: https://code.claude.com/docs/en/setup
- Claude Code authentication: https://code.claude.com/docs/en/authentication
- Maven download and install: https://maven.apache.org/download.cgi and https://maven.apache.org/install
