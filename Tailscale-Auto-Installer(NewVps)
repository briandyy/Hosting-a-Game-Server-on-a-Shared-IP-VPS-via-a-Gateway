#!/bin/bash

# --- Stop on any error ---
set -e

# --- Define your NEW Tailscale Auth Key here ---
# Go to your Tailscale Admin Console -> Auth Keys and generate a new key.
# It is highly recommended to use an ephemeral, one-time-use key.
TS_AUTH_KEY="PASTE_YOUR_NEW_TAILSCALE_AUTH_KEY_HERE"

# --- Check if Auth Key is set ---
if [ "$TS_AUTH_KEY" == "PASTE_YOUR_NEW_TAILSCALE_AUTH_KEY_HERE" ]; then
    echo "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!"
    echo "!!! ERROR: Please edit this script and replace the       !!!"
    echo "!!! placeholder Tailscale Auth Key with a real one.      !!!"
    echo "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!"
    exit 1
fi

echo ">>> [STEP 1/3] Updating and upgrading system packages..."
# Update package lists and upgrade installed packages without any interactive prompts
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get upgrade -y
echo ">>> System packages updated successfully."
echo

echo ">>> [STEP 2/3] Installing Tailscale..."
# Use the official install script which handles different OS versions
curl -fsSL https://tailscale.com/install.sh | sh
echo ">>> Tailscale installed successfully."
echo

echo ">>> [STEP 3/3] Starting and enabling Tailscale..."
# Start tailscale and log in with your auth key.
# This command also ensures Tailscale is enabled to start on boot.
sudo tailscale up --authkey=${TS_AUTH_KEY}
echo ">>> Tailscale is now running and configured to start on boot."
echo

#automatic changing root login cridential for ssh 
echo 'root:changemetoyourpassword' | chpasswd

echo "--------------------------------------------------------"
echo "✅ VPS setup script completed successfully!"
echo
echo "--------------------------------------------------------"
