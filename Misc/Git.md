# Administration & Operations

#### Authentication & Environment Setup
To set up initial access for GitHub, you must generate a Personal Access Token (PAT):

- Navigate to: `Github > Settings > Developer Settings > Personal access tokens > Tokens > Generate new token (classic)`.
- Add a Note (whatever you want), choose an expiration (shorter is more secure), and ensure the `repo` scope is checked before generating.
- Save this token securely; you will use it when prompted to log in alongside your username.

Set up your local Git environment variables:
```bash
git config --global user.name "<username>"
git config --global user.email "<email>"
git config --list                               # Verify the configuration
```

#### Initial Repository Creation
Follow this sequence to initialize a new repository and push it to GitHub:

```bash
echo "STUFF" >> README.md
git init
git add <files>  # Use 'git add .' for all files in dir, or '-A' for all & recursive
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/jell0man/<repo_name>.git
git push -u origin main
```

#### Day-to-Day Commits
For additional commits during regular workflow:

```bash
git add <files>    # Use -A for all changes
git commit -m "message"
git push origin main
```

#### Emergency Recovery Sequence
If your local repository becomes corrupted or detached, use this sequence to safely restore it:

```bash
# 1. Manually copy your files into a new temporary directory as a backup
rd /S .git                              # Delete the local Git database
git init                                # Re-initialize the repository
git remote add origin <github URL>      # Tell Git where the GitHub remote is
git fetch origin                        # Download clean commit objects
git reset --hard origin/main            # Restore the repo to a clean state
git checkout main                       # Create and switch to local branch main
# 2. Manually copy your backed-up files back into the repository
git add .                               # Add all the changes
git commit -m "<reason>"                # Commit the restored changes
git push origin main                    # Push the changes to the remote repository
```


# Offensive Git (Attacks & PrivEsc)
#### Enumeration & Reconnaissance
If you land on a box that uses Git, you can enumerate the repository state and history.

- **View Directory State:** Use `git status` to display the state of the Git working directory.
- **Restore Deleted Files:** If `git status` reveals deleted files, you can restore them using `sudo git restore .` within the git directory. Follow this with `ls -la` to view any newly recovered files.
- **View Commit History:** Use `git log` to show the full commit history.
- **Deep Dive Commits:** Use `git show <commit_id>` to go in-depth on a specific commit.
    - Type `/<search>` to filter for keywords.
    - Press `n` to jump to the next highlighted keyword.
    - Append `| grep <filter>` to see all instances of a keyword in the output.

#### Dumping Remote Repositories (git-dumper)
Use `git-dumper` to download an entire Git repository from a target URL to your local machine.

```bash
python3 -m venv .venv
source .venv/bin/activate 
python3 -m pip install git-dumper

git-dumper <URL> <output_directory>
```

Once dumped, navigate into the output directory and use `git status`, `git log`, and `git show` exactly as if you were in the actual live directory. Be sure to enumerate the raw files as well.

#### Local PrivEsc via Git Modification
Sometimes you may need to modify files within a Git server for privilege escalation (e.g., if a cron job runs a script located within a git-server directory).

**Step 1: Clone the Repository** Clone the directory to a writeable location so you can modify it:
```bash
cd /tmp/      # Or any other writeable location
git clone file:///<git_server_directory>/
```

**Step 2: Modify Files** Use `echo`, `vim`, or any text editor to make your malicious changes to the files.
```bash
chmod 777 <file>   # Ensure all users can access the modified file
```

**Step 3: Commit the Changes** Add and commit your changes. If you encounter an "unknown author" error, configure a temporary identity:
```bash
git add <file>     # OR git add -A
git commit -m "<any_name_you_want>"

# Fix unknown author errors if they appear:
git config --global user.name "<name_we_want>"
git config --global user.email "<email_we_want>"
```

**Step 4: Push to Master** Push your poisoned files back to the master branch to execute the attack:
```bash
git push origin master
```

#### Git Commands over SSH
If you need to make changes to a remote Git server from your local attacking machine, you can pass custom SSH parameters (like a private key or non-standard port) directly to Git.

**Clone over SSH:**
```bash
GIT_SSH_COMMAND='ssh -i <path/to/key> -p <port>' git clone <user>@<victim_ip>:/path/to/<git-server>
```

**Push over SSH:**
```bash
GIT_SSH_COMMAND='ssh -i <path/to/key> -p <port>' git push origin master
```

_Note: Aside from wrapping the commands in `GIT_SSH_COMMAND`, all other Git modifications follow the local PrivEsc steps outlined above._
