# CachyGameModeUpdateGUI 🎮

A streamlined, touch-and-gamepad-friendly graphical interface for the official `cachy-update` tool. 

Designed specifically for **CachyOS Handheld Edition** devices (like the Lenovo Legion Go, ROG Ally, and Steam Deck), this updater eliminates the need for a physical keyboard. It provides a massive, easily readable UI that feels native to handheld gaming environments, seamlessly integrating into your desktop menu and Steam Gaming Mode. Built with PyQt6, the application utilizes native hardware rendering for crisp text, smooth drop shadows, and flawless integration with KDE Plasma and standard Linux compositors.

## ✨ Features
* **Handheld Optimized:** Large, touch-friendly buttons, modern matte-glass aesthetic, and scalable UI elements mapped directly to gamepad controls (A/Enter to update, Y to toggle terminal, X to close).
* **Secure Memory Vault:** Enter your password once at the start of the update. The app securely holds it in RAM using Python's `pty` module to automatically handle terminal prompts, then flushes it the moment the update completes—zero `visudo` hacks required.
* **Smart Auto-Responder:** Automatically handles routine mid-update prompts (e.g., confirming package installations and safely restarting `snapperd` services) without requiring user intervention.
* **Live Terminal Output:** Keep it simple, or expand the console view to see exactly what `pacman` and `paru` are doing under the hood.
* **Automated Setup:** A single script handles dependencies, generates a custom handheld-themed SVG icon, and perfectly integrates the app into your application menu and dock.

---

## 🚀 Installation & Setup

You do not need to manually install dependencies or edit system shortcuts. The automated installer does everything for you. Download or clone this repository, open a terminal inside the folder, and run this single command block (do not run with `sudo`; the script will prompt you when needed):

```bash
chmod +x install.sh uninstall.sh && ./install.sh
```

Once finished, **System Updater (Secure)** will instantly appear in your application menu with its custom gamepad icon. You can also easily add it to Steam as a Non-Steam Game to run it straight from Gaming Mode.

---

## 🗑️ Uninstallation

Want to revert to standard terminal updates? The uninstaller scrubs everything cleanly, removing the application files, desktop shortcuts, custom icons, configuration data, and the PyQt6 GUI library dependency. Run this single command block from the repository folder:

```bash
./uninstall.sh
```

---

## 🔍 Security Audit / Source Code

For transparency, the underlying Python application that the `install.sh` script generates and deploys is provided below. It strictly relies on the standard `cachy-update` command and uses `pty` to securely handle sudo authentication in memory without altering `/etc/sudoers`.

```python
#!/usr/bin/env python
import sys
import subprocess
import threading
import pty
import os
import re
import fcntl
import termios
import json
from pathlib import Path

from PyQt6.QtWidgets import (QApplication, QWidget, QVBoxLayout, QHBoxLayout, 
                             QLabel, QLineEdit, QPushButton, QProgressBar, 
                             QTextEdit, QFrame, QGraphicsDropShadowEffect, QCheckBox)
from PyQt6.QtCore import Qt, pyqtSignal, QObject, QTimer
from PyQt6.QtGui import QColor, QFont, QTextCursor

# Setup configuration path
CONFIG_DIR = Path.home() / ".config" / "cachy-updater"
CONFIG_FILE = CONFIG_DIR / "config.json"

def load_config():
    if CONFIG_FILE.exists():
        try:
            with open(CONFIG_FILE, "r") as f:
                return json.load(f)
        except Exception:
            pass
    return {"theme": "Dark", "auto_close": False}

def save_config(config):
    CONFIG_DIR.mkdir(parents=True, exist_ok=True)
    with open(CONFIG_FILE, "w") as f:
        json.dump(config, f)

# Professional Desktop Color Palette
THEME_COLORS = {
    "Dark": {
        "bg": "#0A0E14",            
        "card": "#111620",          
        "btn_grad_1": "#0DB9D7",
        "btn_grad_2": "#088DA5",
        "btn_hover_1": "#12CEEE",
        "btn_hover_2": "#099EBA",
        "text": "#E2E8F0",          
        "text_sub": "#8A99B5",      
        "input_bg": "#07090D",      
        "border": "rgba(255, 255, 255, 0.08)",        
        "border_focus": "#0DB9D7",
        "prog_bg": "#1A212F"
    },
    "Light": {
        "bg": "#F1F5F9", "card": "#FFFFFF", "btn_grad_1": "#057A91", "btn_grad_2": "#045C6E", 
        "btn_hover_1": "#068FA8", "btn_hover_2": "#056D82", "text": "#1E293B", "text_sub": "#64748B", 
        "input_bg": "#F8FAFC", "border": "rgba(0, 0, 0, 0.1)", "border_focus": "#057A91", "prog_bg": "#E2E8F0"
    }
}

class UpdaterSignals(
