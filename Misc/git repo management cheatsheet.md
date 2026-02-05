
Set up initial access for github.
```powershell
# Generate PAT (personal access token)
Github > Settings > Developer Settings > Personal access tokens > Tokens > Generate new token (classic)
Note -- whatever you want
Expiration # whatever you want, shorter is more secure
Scopes # repo is sufficient
Generate Token

# save token
# Use git as normal, you will be prompted to login with user and PAT
```

Set up git env
```powershell
git config --global user.name "<username>"
git config --global user.email "<email>"

#verify
git config --list
```

initial repo creation
```powershell
echo "STUFF" >> README.md
git init
git add <files>  # git add . for all files in dir / -A = all & recursive
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/jell0man/<repo_name>.git
git push -u origin main
```

additional commits
```powershell
git add <files>    # -A for all
git commit -m "message"
git push origin main
```

Recovery Sequence (In case repo gets messed up)
```powershell
Manually copy files into new directory  # Backup repo
rd /S .git                              # Delete Git database
git init                                # Re-initialize repo
git remote add origin <github URL>      # Tell Git where Github is
git fetch origin                        # Download clean commit objects
git reset --hard origin/main            # Restore repo to clean state
git checkout main                       # Create local branch main
Manually copy files back into repo      # Moving files back into repo
git add .                               # Add all changes
git commit -m "<reason>"                # Commit changes
git push origin main                    # Push changes to remote repo
```