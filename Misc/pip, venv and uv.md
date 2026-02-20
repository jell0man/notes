When running scripts and to avoid dependency hell, don't install dependencies globally. Use virtual environments to install dependencies instead.

Use `uv` -- faster, better version management, strict resolution. Use `venv` when living off the land.

Using `venv` (Standard library)
```bash
# Setup
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt

# Run your script
python3 script.py

# Leave the environment
deactivate
```

Using `uv` (modern alternative)
```bash
# install uv globally
curl -LsSf https://astral.sh/uv/install.sh | sh   

# Setup
uv venv [--python 3.9]   # python version optional... good if you need legacy stuff
source .venv/bin/activate
uv pip install -r requirements.txt

# Run your script
python3 script.py

# Leave the environment
deactivate
```


