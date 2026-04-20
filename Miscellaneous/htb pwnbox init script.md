```bash
#!/bin/bash
#This script is executed every time your instance is spawned.

set -euo pipefail

# Config
VAULT_NAME="${VAULT_NAME:-Methodology}"
VAULT_PARENT="${VAULT_PARENT:-$HOME/Obsidian}"
VAULT_PATH="${VAULT_PARENT}/${VAULT_NAME}"
APPIMAGE_URL="https://github.com/obsidianmd/obsidian-releases/releases/download/v1.10.3/Obsidian-1.10.3.AppImage"
APPIMAGE_DEST="$HOME/.local/bin/obsidian"

# Create directories
git clone https://github.com/jell0man/methodology_notes $VAULT_PATH
mkdir "$HOME/.local/bin"

# Set up binary
curl -L "$APPIMAGE_URL" -o "$APPIMAGE_DEST"
chmod +x "$APPIMAGE_DEST"

# Customize appearance
mkdir "$VAULT_PATH/.obsidian"
echo '{
  "theme": "obsidian",
  "baseTheme": "dark"
}' >> "$VAULT_PATH/.obsidian/appearance.json"

```