# Gateway-Proxy-for-vps-with-port-forwarding-need-2-vps-
Gateway Proxy for vps with port forwarding (need 2 vps)

## **Guide: Hosting a Game Server on a Shared IP VPS via a Gateway**

This guide explains the standard method for hosting a game server (like Left 4 Dead 2 or Minecraft Bedrock) that requires a direct IP and open ports when your main server is on a shared IP address.

### **The Concept**

We use two servers to solve the "shared IP" problem:

1.  **GAME Server**: Your primary VPS with a **shared IP**. This is where you will install and run the actual game server software (L4D2, Minecraft, etc.).
2.  **GATEWAY Server**: A second, cheap or free VPS (like from Oracle/Azure) that has its own dedicated **Public IP Address**. This server's only job is to catch traffic from the internet and forward it to the GAME server.

The two servers will be connected by a secure, private network using **Tailscale**.

-----

### **Prerequisites**

Before you begin, make sure you have:

  * ✅ Login access (via SSH) to both your GAME Server and your GATEWAY Server.
  * ✅ A free [Tailscale account](https://login.tailscale.com/).

-----

### **Part 1: Initial Setup & Private Networking**

The first goal is to get both servers talking to each other on a private, secure network.

#### **Step 1.1: Generate a Tailscale Auth Key**

Using an Auth Key is much easier than logging in via a browser for servers.

1.  Go to the **[Keys page](https://login.tailscale.com/admin/settings/keys)** in your Tailscale Admin Console.
2.  Click **"Generate auth key..."**.
3.  Give it a description like "Game Server Key".
4.  Make sure the key is **NOT** "Ephemeral".
5.  Click **"Generate key"**.
6.  **Copy the key** (`tskey-auth-...`) and save it somewhere safe. You will need it twice.

#### **Step 1.2: Configure the GAME Server**

Connect to your Shared IP VPS and run these commands.

```bash
# Update your server and install Tailscale
sudo apt update && sudo apt upgrade -y
curl -sSL https://tailscale.com/install.sh | sh

# Connect to your Tailscale network using your secret Auth Key
# Replace the placeholder with your real key
sudo tailscale up --authkey <YOUR_TAILSCALE_AUTH_KEY>
```

#### **Step 1.3: Configure the GATEWAY Server**

Now, connect to your free Oracle/Azure VPS and do the same thing.

```bash
# Update your server and install Tailscale
sudo apt update && sudo apt upgrade -y
curl -sSL https://tailscale.com/install.sh | sh

# Connect to your Tailscale network using the SAME Auth Key
# Replace the placeholder with your real key
sudo tailscale up --authkey <YOUR_TAILSCALE_AUTH_KEY>
```

#### **Step 1.4: Verify the Connection**

On either server, run `tailscale status`. You should see both of your servers listed with their private `100.x.y.z` IP addresses. This confirms they can see each other. Congratulations, your private network is active\!

-----

### **Part 2: Configuring the GAME Server**

Now, let's install the game on your main shared IP VPS.

#### **Step 2.1: Install Game Software**

This example is for **Left 4 Dead 2**. For other games, follow the appropriate installation guide.

```bash
# Connect to your GAME Server via SSH
# Enable 32-bit support and install steamcmd
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install steamcmd -y

# Download the L4D2 server files
mkdir ~/l4d2_server && cd ~/l4d2_server
steamcmd +force_install_dir ./l4d2 +login anonymous +app_update 222860 validate +quit
```

#### **Step 2.2: Run the Game Server**

Start the server so it's ready to accept connections from the gateway.

```bash
# Navigate to the game directory
cd ~/l4d2_server/l4d2

# Run the server (you can use 'screen' to keep it running)
./srcds_run -game left4dead2 +map c1m1_hotel
```

-----

### **Part 3: Configuring the GATEWAY Server**

This is the final and most important configuration.

#### **Step 3.1: Open Cloud Firewall Ports**

This is a critical step. Go to your **Oracle or Azure dashboard** for your Gateway VM. In the **Networking / Firewall** settings, add **Ingress (Incoming) Rules** to allow traffic for your game.

  * **For Left 4 Dead 2:**

      * Allow Protocol **TCP** on Port **`27015`**
      * Allow Protocol **UDP** on Port **`27015`**

  * **For Minecraft Bedrock:**

      * Allow Protocol **UDP** on Port **`19132`**

#### **Step 3.2: Install Forwarding Tools**

We need `socat` (our reliable forwarder) and `screen` (to keep it running).

```bash
# Connect to your GATEWAY Server via SSH
sudo apt update && sudo apt install socat screen -y
```

#### **Step 3.3: Create and Run the Forwarding Script**

Using a script makes this process easier and reusable.

1.  **Find your GAME Server's Tailscale IP**. Run `tailscale status` and find the `100.x.y.z` IP address for your Game Server.

2.  **Create the script file** on your **GATEWAY server**:

    ```bash
    nano run_forwarder.sh
    ```

3.  **Copy and paste the code below** into the `nano` editor. **Change the `GAME_SERVER_IP` and `GAME_PORT` variables** to match your setup.

    ```bash
    #!/bin/bash
    #
    # A simple script to forward game server traffic using socat.
    #

    # --- CONFIGURATION ---
    # 1. Set the Tailscale IP of your GAME server. Find this with 'tailscale status'.
    GAME_SERVER_IP="100.x.y.z"

    # 2. Set the port you want to forward. (e.g., "27015" for L4D2, "19132" for MC Bedrock)
    GAME_PORT="27015"
    # --- END CONFIGURATION ---

    echo "Starting TCP forwarder for port $GAME_PORT..."
    socat TCP4-LISTEN:$GAME_PORT,fork TCP4:$GAME_SERVER_IP:$GAME_PORT &

    echo "Starting UDP forwarder for port $GAME_PORT..."
    socat UDP4-LISTEN:$GAME_PORT,fork UDP4:$GAME_SERVER_IP:$GAME_PORT &

    echo "Forwarders are running for TCP and UDP on port $GAME_PORT."
    echo "Forwarding to $GAME_SERVER_IP"
    wait
    ```

    *Note: For a UDP-only game like Minecraft Bedrock, you can delete the `socat TCP4...` line.*

4.  **Save and exit `nano`**: Press `Ctrl+X`, then `Y`, then `Enter`.

5.  **Make the script executable**:

    ```bash
    chmod +x run_forwarder.sh
    ```

6.  **Run the script inside `screen`** to keep it running forever:

    ```bash
    # Start a new screen session
    screen

    # Run your new script
    ./run_forwarder.sh

    # Detach from the screen (the script will keep running)
    # Press Ctrl+A, release the keys, then press D.
    ```

-----

### **Part 4: Connecting to Your Server**

You are done\! The IP address you share with your friends is the **Public IP of your GATEWAY Server**.

  * **For Left 4 Dead 2:**

      * Open the console and type: `connect <YOUR_GATEWAY_PUBLIC_IP>:27015`

  * **For Minecraft Bedrock:**

      * Add a new server with:
      * **Server Address:** `<YOUR_GATEWAY_PUBLIC_IP>`
      * **Port:** `19132`

Congratulations\! You have successfully built a reliable and powerful game server gateway.
