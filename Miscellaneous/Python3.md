Common Imports
```python
import sys, os          # OS interaction
import requests         # Web exploitation/APIs
import json             # Parsing logs/results
import subprocess       # Running system commands (nmap, etc.)
import socket           # Low-level networking
import argparse         # Professional CLI flags
```

Variables and Typing
```python
# Basic types
name = "jell0man"          # String
count = 10                 # Integer
pi = 3.14                  # Float
is_active = True           # Boolean

# Type Casting
string_num = str(100)
integer_val = int("50")

# F-Strings (Best for output/logging)
print(f"User: {name} | Active: {is_active}")
```

Computing and Comparisons
```python
# Arithmetic
result = 10 / 3    # Float division (3.33)
result = 10 // 3   # Floor division (3)
remainder = 10 % 3 # Modulo (1)
power = 2 ** 3     # 8

# Comparisons / Operators
# ==, !=, >, <, >=, <=
if name == "root" or is_active:
    print("Access Granted")

# Membership testing (Great for lists/sets)
if "root" in ["admin", "root", "operator"]:
    print("Found")
```

Command Line Arguments
```python
# Use sys.argv for quick and dirty scripts, or argparse for professional tools

import sys

# sys.argv[0] is the script name
# sys.argv[1] is the first argument
if len(sys.argv) < 2:
    print("Usage: python3 script.py <target_ip>")
    sys.exit(1)

target = sys.argv[1]
```

Functions
```python
def pwn_check(target, port=80):
    """Docstring explaining the function"""
    if target == "127.0.0.1":
        return "Localhost skipped"
    return f"Scanning {target} on {port}..."

# Calling
msg = pwn_check("10.10.10.1", port=443)
```

Loops
```python
# Range loop
for i in range(5):
    print(f"Attempt {i}")

# List/Iterator loop
targets = ["10.0.0.1", "10.0.0.2"]
for ip in targets:
    print(f"Pinging {ip}")

# While loop
count = 0
while count < 3:
    print("Checking...")
    count += 1
```

Lists
```python
ips = ["127.0.0.1", "192.168.1.1"]
ips.append("10.10.10.10")    # Add to end
ips.pop()                    # Remove last item
first_ip = ips[0]            # Access
subset = ips[1:3]            # Slicing
```

Dictionaries
```python
# Creation
user_data = {"username": "jell0man", "uid": 1000}

# Access/Update
print(user_data["username"])
user_data["shell"] = "/bin/zsh"

# Safe access (prevents KeyError)
shell = user_data.get("shell", "/bin/sh")
```

Sets
```python
# Unique and unorderd - Perfect for removing duplicates from lists

discovered_hosts = {"10.0.0.1", "10.0.0.2", "10.0.0.1"}
# Result: {'10.0.0.1', '10.0.0.2'}

discovered_hosts.add("10.0.0.5")
```

Errors and Exceptions
```python
try:
    with open("targets.txt", "r") as f:
        data = f.read()
except FileNotFoundError:
    print("[-] Error: targets.txt not found.")
except Exception as e:
    print(f"[-] An unexpected error occurred: {e}")
finally:
    print("[*] Script cleanup finished.")
```

Classes and Initialization
```python
class Target:
    def __init__(self, ip, hostname="Unknown"):
        self.ip = ip              # Public attribute
        self.hostname = hostname  # Public attribute
        self.__secret_key = None  # Private attribute (internal use only)

    def display(self):
        print(f"Target: {self.hostname} ({self.ip})")

# Usage
machine = Target("10.10.10.5", "DC01")
machine.display()
```

Inheritance
```python
class Scanner:
    def run(self):
        print("Starting generic scan...")

class PortScanner(Scanner):
    def run(self): # Overriding the parent method
        print("Starting Nmap port scan...")

class WebScanner(Scanner):
    def run(self):
        print("Starting Directory Bruteforce...")

# Polymorphism in action
tools = [PortScanner(), WebScanner()]
for tool in tools:
    tool.run()
```
