### **The Full Tutorial: Automatic Server Backups to Google Drive**

#### **Part 1: Install the Tools**

First, we log in as `root` and install all the necessary programs.

**Action:** Run these two commands on your VPS.

```bash
# 1. Update your server and install the zip and curl programs
apt update && apt install zip curl -y

# 2. Install rclone (the program for Google Drive)
curl https://rclone.org/install.sh | bash
```

-----

#### **Part 2: Connect to Your Google Drive**

Next, we will authorize your VPS to securely access your Google Drive account.

**Action:** Start the `rclone` setup process.

```bash
rclone config
```

Follow these prompts exactly.

1.  `n)` (to create a **n**ew remote)
2.  **name\>**: Call it `gdrive_backup`
3.  **Storage\>**: A list of cloud services will appear. Find "Google Drive" and type its number, then press Enter.
4.  **client\_id\>**: Press **Enter** (leave it blank).
5.  **client\_secret\>**: Press **Enter** (leave it blank).
6.  **scope\>**: Type `1` for full access.
7.  **root\_folder\_id\>**: Press **Enter**.
8.  **service\_account\_file\>**: Press **Enter**.
9.  **Edit advanced config?\>**: Type `n` for No.
10. **Use auto config?\>**: **Type `n` for No.** (This is the most important step on a server).

**Authorization Step:**

  * Your terminal will show you a URL like `https://accounts.google.com/o/oauth2/auth?....`
  * **Copy** this full URL.
  * **Paste** it into a web browser on your phone or personal computer.
  * Log in to the Google account you want to use for backups and click "Allow".
  * Google will show you a **verification code**. Copy this code.
  * Go back to your VPS terminal and **paste the code**, then press Enter.

**Finalizing Setup:**

  * Confirm `y)` (Yes, this is OK).
  * Quit `q)`.

**Action:** Test the connection to make sure it works.

```bash
rclone lsd gdrive_backup:
```

This command should list the folders in your Google Drive. If it does, you are ready for the next step.

-----

#### **Part 3: Create the Backup Script**

Now we create the main script file that will contain all the instructions for the backup.

**Action:** Use the `nano` text editor to create a file named `backup.sh` in your root directory.

```bash
nano /root/backup.sh
```

**Action:** Copy the entire code from the box below and paste it into the `nano` window.

```sh
#!/bin/bash

# =================================================================
# FULL AUTOMATED BACKUP SCRIPT (ROOT USER METHOD)
# Final Version - For mcbserver folder
# Filename Format: "Backup - Dayname Day Month Year.zip"
# =================================================================

### --- Main Settings --- ###

# The directory to back up.
SERVER_DIR="/root/mcbserver"

# A temporary directory to store the zip file before uploading.
BACKUP_DIR="/root/temp_backups"

# The rclone remote name you configured in Part 2.
GDRIVE_REMOTE="gdrive_backup"

# The folder inside Google Drive to store the backups.
GDRIVE_DIR="ServerBackups/mcbserver"

# Number of days to keep backups on Google Drive.
DAYS_TO_KEEP=7

# --- Script Logic ---

echo "---"
echo "Starting Backup for $(date +"%A, %d %B %Y at %r")"

# Check if the server folder actually exists
if [ ! -d "$SERVER_DIR" ]; then
  echo "Error: Server directory $SERVER_DIR does not exist. Aborting."
  exit 1
fi

# Create the unique filename based on your requested format.
# WARNING: This format contains spaces. The script uses quotes to handle it,
# but this is generally not recommended. Do not remove the quotes.
TIMESTAMP=$(date +"%A %d %B %Y") 
FILENAME="Backup - $TIMESTAMP.zip"
FULL_PATH_TO_ZIP="$BACKUP_DIR/$FILENAME"

# 1. Create the temporary backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"
echo "Step 1: Compressing server files into \"$FILENAME\"..."

# 2. Create the compressed .zip file (variable is in quotes to handle spaces)
cd "$SERVER_DIR" || exit
zip -r "$FULL_PATH_TO_ZIP" . > /dev/null
cd - > /dev/null

# 3. Upload the .zip file to Google Drive (variable is in quotes)
echo "Step 2: Uploading to Google Drive..."
rclone copy "$FULL_PATH_TO_ZIP" "$GDRIVE_REMOTE:$GDRIVE_DIR" --progress

# 4. Delete the temporary .zip file from the VPS (variable is in quotes)
echo "Step 3: Cleaning up local temporary file..."
rm "$FULL_PATH_TO_ZIP"

# 5. Delete backups on Google Drive that are older than DAYS_TO_KEEP
echo "Step 4: Cleaning up old backups on Google Drive..."
rclone delete --min-age ${DAYS_TO_KEEP}d "$GDRIVE_REMOTE:$GDRIVE_DIR"

echo "Backup Complete!"
echo "---"
```

**Action:** Save the script and exit the editor by pressing `Ctrl+X`, then `Y`, then `Enter`.

**Action:** Make the script runnable.

```bash
chmod +x /root/backup.sh
```

-----

#### **Part 4: Automate the Script**

This last step will schedule your script to run every morning automatically.

**Action:** Open the system scheduler (crontab).

```bash
crontab -e
```

(If it asks you to choose an editor, pick `nano` by typing `1`).

**Action:** Go to the very bottom of the file and paste this one line:

```crontab
# Run the server backup script every day at 4:05 AM
5 4 * * * /root/backup.sh > /root/backup.log 2>&1
```

This schedule will run your script at 4:05 AM every morning (based on your server's clock). All output will be saved to `/root/backup.log` so you can check it later.

**Action:** Save and exit the scheduler (`Ctrl+X`, `Y`, `Enter`).

-----

### **Congratulations, You Are Done\!**

Your automatic backup system is now fully configured and active.

  * **To test it right now**, you can run the script manually: `/root/backup.sh`. Watch the output, and then check your Google Drive for the new file.
  * **Tomorrow morning (Friday, August 15th)**, at 4:05 AM your time, the system will run by itself. The backup file will appear in Google Drive with the name: **`Backup - Friday 15 August 2025.zip`**.
