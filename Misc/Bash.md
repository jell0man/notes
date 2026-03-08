Shortcuts
```Bash
TAB            # autocomplete
Ctrl + r       # search command history
Ctrl + a / e   # move cursor to start / end of line
Ctrl + u / k   # cut from cursor to start / end of line
!!             # repeat last command
!$             # use last argument of previous command
```

Common Commands & Utilities
```Bash
$VAR           # access variable
printenv       # list all environment variables
clear          # clear screen
alias          # list or create aliases (alias ll='ls -la')
ls -R          # list files recursively
grep -i        # search text (case-insensitive)
awk '{print $1}' # extract first column of output
sed 's/old/new/g' # find and replace text
find / -name   # search for files by name
```

Operators
```Bash
# Comparison (Integers)
-eq            # equal to
-ne            # not equal to
-lt / -gt      # less than / greater than
-le / -ge      # less than or equal / greater than or equal

# Comparison (Strings)
==             # equal to
!=             # not equal to
-z             # string is empty
-n             # string is not empty

# Logical
&&             # AND
||             # OR
!              # NOT

# Redirection
>              # overwrite file
>>             # append to file
2>             # redirect error stream
2>&1           # merge error and success streams
|              # pipe output to next command
```

Variables and Typing
```Bash
# No spaces around the '=' sign
name="jell0man"           # String
count=10                  # Integer
is_active=true            # Boolean (convention)

# Environment Variables
export TARGET="10.10.10.1"

# Command Substitution
current_dir=$(pwd)

# Basic Math (Arithmetic Expansion)
result=$((10 + 5))
```

Computing and Comparisons
```Bash
# Double brackets [[ ]] are preferred in Bash for tests
if [[ $name == "root" || $is_active == true ]]; then
    echo "Access Granted"
fi

# Checking if a file exists
if [[ -f "/etc/passwd" ]]; then
    echo "File exists"
fi
```

Command Line Arguments
```Bash
# $0 is the script name
# $1, $2... are positional arguments
# $# is the number of arguments

if [[ $# -lt 1 ]]; then
    echo "Usage: $0 <target_ip>"
    exit 1
fi

target=$1
```

Functions
```Bash
pwn_check() {
    local target=$1 # 'local' keeps variable scoped to function
    local port=${2:-80} # default to 80 if $2 is empty

    if [[ $target == "127.0.0.1" ]]; then
        echo "Localhost skipped"
        return
    fi
    echo "Scanning $target on $port..."
}

# Calling (No parentheses)
pwn_check "10.10.10.1" 443
```

Loops
```Bash
# Range loop
for i in {1..5}; do
    echo "Attempt $i"
done

# List loop
targets=("10.0.0.1" "10.0.0.2")
for ip in "${targets[@]}"; do
    echo "Pinging $ip"
done

# While loop
count=0
while [[ $count -lt 3 ]]; do
    echo "Checking..."
    ((count++))
done
```

Arrays
```Bash
ips=("127.0.0.1" "192.168.1.1")
ips+=("10.10.10.10")      # Add item
echo ${ips[0]}            # Access first item
echo ${#ips[@]}           # Length of array
```

Errors and Exceptions
```Bash
# Bash doesn't have try/catch; check exit codes ($?) instead
cp source.txt dest.txt 2>/dev/null

if [[ $? -ne 0 ]]; then
    echo "[-] Error: Copy failed."
    exit 1
fi

# Strict Mode (Safe Scripting)
set -e # exit on error
set -u # exit on unset variables
set -o pipefail # exit if any part of a pipe fails
```