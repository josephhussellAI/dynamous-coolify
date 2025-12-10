# The Beelink NAS PaaS: A Complete HomeLab Setup Guide

**Architecture:** Ubuntu Server (eMMC Boot) | 2x 1TB RAID 1 (Data) | Tailscale (Mgmt) | Cloudflare Tunnel (Public)

> [!NOTE]
> **Why eMMC?** You asked about disadvantages. eMMC is slower and has less endurance than SSDs.
> **The Mitigation:** This guide configures the system to run the *OS* on the eMMC (low write intensity) but moves all *Docker Containers and Data* to your RAID 1 SSDs (high speed/endurance). You get the best of both worlds: saved slots AND high performance.

---

## � Prerequisites
Before you begin, ensure you have the following:

### Hardware (Required)
*   **Beelink ME Mini NAS** (or similar N100 hardware).
*   **512GB NVMe Drive** (This usually comes pre-installed).
*   **Ethernet Cable** (For stable internet during setup).
*   **Monitor & Keyboard** (Just for the first 15 minutes).

### Hardware (Optional - For RAID)
*   **2x 1TB SATA SSDs** (For the RAID 1 Mirror).
*   *Note: If you only use the single NVMe drive, skip the RAID steps in this guide.*

### Software & Accounts
*   **Ubuntu Server 24.04 LTS USB**: [Download Here](https://ubuntu.com/download/server) and flash to USB.
*   **Tailscale Account**: [tailscale.com](https://tailscale.com) (Free Personal version is sufficient).
*   **Cloudflare Account**: [cloudflare.com](https://cloudflare.com) (Free Tier is sufficient).
*   **A Domain Name**: You must own a domain (e.g., `yourdomain.com`).
    *   **Cost:** This is the *only* thing you need to buy. You can register one on Cloudflare for ~$9/year.
    *   *Note: This guide assumes your domain is managed by Cloudflare.*
    *   If you bought your domain elsewhere (GoDaddy, Namecheap, etc.), you must [point your nameservers to Cloudflare](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/) before starting. We do not cover other DNS setups in this guide.

---

**Architecture:** Ubuntu Server (eMMC Boot) | 2x 1TB RAID 1 (Data) | Tailscale (Mgmt) | Cloudflare Tunnel (Public)

> [!NOTE]
> **Why eMMC?** This guide is tailored for the **Beelink ME Mini NAS** (and similar servers) which include a small eMMC storage chip. We configure the OS to run here to save your NVMe/SATA slots for high-speed data.
> **Not on Beelink?** If your server doesn't have eMMC, don't worry! simply follow **Option A** (Single Drive) in Phase 1 to install everything on your main drive. The rest of the guide works exactly the same.

---

## �🛑 Phase 1: Installing Ubuntu (The Easy Way)
*Goal: Get the OS running on the internal chip without touching your big drives yet.*

1.  **Boot the Installer:**
    *   Insert your Ubuntu Server 24.04 USB.
    *   Tap `F7` (or `Del`) to boot from USB.
2.  **Guided Storage Config:**
    *   When asked about storage, you have two paths:
    
    #### Option A: Single Drive (Simple)
    *   If you are **NOT** doing RAID and just using the included drive:
    *   Select "Use an entire disk".
    *   Choose the **512GB NVMe**.
    *   Done.

    #### Option B: RAID 1 + eMMC (Advanced)
    *   When asked about storage, **uncheck** "Set up this disk as an LVM group" (keep it simple).
    *   Select the **64GB eMMC (mmcblk0)** as the install target.
    *   **IMPORTANT:** Leave your two 1TB drives **unchecked/unformatted**. We will set them up safely in Phase 3.
3.  **Profile:**
    *   **Name:** `admin`
    *   **Server name:** `beelink-nas`
    *   **Username:** `user` (or whatever you prefer)
    *   **Password:** Choose a strong password. We recommend using the **Token Generator tool** at [it-tools.tech](https://it-tools.tech/). Generate a **64-character** password using letters (uppercase/lowercase), numbers, and symbols. *(Fun Fact: We will be self-hosting this exact tool on your NAS later!)*.
    *   **SSH:** Select `[x] Install OpenSSH server`.
4.  **Finish:** Install and Reboot.

---

## 🔒 Phase 2: Secure Management Access (Tailscale VPN)
*Goal: Create a secure "backdoor" so you can access your server from anywhere, and set up a stable internal hostname (MagicDNS).*

1.  **Log in** to your new server (physically or via local SSH).
2.  **Install Tailscale:**
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    ```
3.  **Authenticate (With SSH enabled):**
    ```bash
    sudo tailscale up --ssh
    ```
    *   Click the link to authorize with your Tailscale account.

### 4. Configure MagicDNS & Reliability (Critical)
We need a stable name for your server so Coolify (and you) can always find it.

1.  **Disable Key Expiry (Crucial for Servers):**
    *   Go to **Tailscale Admin Console** -> **Machines**.
    *   Find `beelink-nas` -> Click **...** (menu) -> **Disable Key Expiry**.
    *   *Why?* Standard keys expire every 90 days. If this expires, you lose access.
2.  **Get your MagicDNS Name:**
    *   In the Admin Console, verify **DNS** -> **MagicDNS** is **Enabled**.
    *   Click on your `beelink-nas` machine to see its details.
    *   Find the **Full Domain Name** (e.g., `beelink-nas.happy-hippo.ts.net`).
    *   **COPY THIS.** This is your server's permanent internal address. We will use it for secure tunneling.
3.  **Verify:**
    *   Open a terminal on your laptop (must have Tailscale installed).
    *   Run: `ssh user@beelink-nas.happy-hippo.ts.net` (replace with your name).
    *   If it connects, your internal DNS is working!

---

## ⚙️ Phase 3: Performance & Storage (The "Heavy Lifting")
*Goal: Tune the system and configure storage.*

### 1. Build the Mirror (RAID 1)
**🛑 IF YOU ARE USING A SINGLE DRIVE (Option A): SKIP THIS STEP.**

Combine your two 1TB drives into one fast, redundant space.

```bash
# Update and get tools
sudo apt update && sudo apt install mdadm -y

# ⚠️ Verify drive names (usually sda and sdb)
lsblk

# Create the Mirror (RAID 1)
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb
# Type 'y' if prompted.

# Format as EXT4 (Fastest for 12GB RAM)
sudo mkfs.ext4 -F /dev/md0

# Create mount point and mount
sudo mkdir -p /mnt/data
sudo mount /dev/md0 /mnt/data

# Make it permanent
echo '/dev/md0 /mnt/data ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

### 2. Tuning for n8n & Speed
Apply kernel fixes for high-traffic networks. **(Everyone do this)**.

```bash
# Network & Process Tuning
echo "net.core.rmem_max=2500000" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3. Move Docker to RAID (CRITICAL)
**🛑 IF YOU ARE USING A SINGLE DRIVE (Option A): SKIP THIS STEP.**
This steps saves your eMMC from burning out.

```bash
# 1. Create the Docker directory on your fast RAID drive
sudo mkdir -p /mnt/data/docker

# 2. Tell Docker to store EVERYTHING there (and fix VPN packet size)
# This explicitly prevents Docker from using your eMMC boot drive.
echo '{"data-root": "/mnt/data/docker", "mtu": 1280}' | sudo tee /etc/docker/daemon.json

# 3. Apply changes (If Docker is already installed)
# Note: If this says "Unit docker.service not loaded", ignore it. It just means Docker isn't installed yet.
sudo systemctl restart docker || true
```
**IF YOU ARE SINGLE DRIVE:** You still need the MTU fix for VPNs, just run this simpler command:
```bash
echo '{"mtu": 1280}' | sudo tee /etc/docker/daemon.json
```

## 🌩️ Phase 4: Coolify & The Tunnel ("All Resource" Method)
*Goal: Install Coolify and establish the Cloudflare Tunnel using the official Coolify integration.*

### 1. Install Coolify
```bash
# Install Coolify (It will detect our Docker config on /mnt/data)
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```
*   Wait a few minutes.
*   Open your browser and go to `http://100.x.y.z:8000` (Your Tailscale IP).
*   Create your Admin account.

### 2. Create Tunnel (Cloudflare Dashboard)
1.  Go to **Zero Trust** -> **Networks** -> **Tunnels**.
2.  Click **Add a Tunnel** -> **Select Cloudflared**.
3.  Name: `beelink-nas`.
4.  **Save the Token:** You will see a command like `cloudflared service install eyJh...`.
    *   **COPY ONLY THE TOKEN** (The long string starting with `eyJh...`).
    *   Do *not* run the command on your server. We will let Coolify handle it.

### 3. Deploy Tunnel (Inside Coolify)
1.  In Coolify, go to **Projects**.
2.  Create a project named **"System and Admin Tools"**.
    *   *Note: We will use the default "Production" environment for this system service.*
3.  Click **+ New** -> **Service** -> Search for **"Cloudflared"**.
4.  Click **Configure**.
5.  In **Environment Variables**, paste your **Tunnel Token** into `TUNNEL_TOKEN`.
6.  **Deploy**.
    *   *Result:* Your tunnel is now running as a container managed by Coolify!

---

## 🔒 Phase 5: Upgrade to Full HTTPS (The "Full TLS" Merge)
*Goal: Now that the tunnel is running, we lock it down with End-End Encryption.*

### 1. Generate Origin Certificates (Cloudflare)
1.  Go to **Cloudflare Dashboard** -> **SSL/TLS** -> **Origin Server**.
2.  Click **Create Certificate**.
    *   **Hostnames:** `yourdomain.com` and `*.yourdomain.com`.
    *   **Validity:** 15 Years.
3.  Click **Create**.
    *   **Save Key:** Copy the "Private Key".
    *   **Save Cert:** Copy the "Origin Certificate".

### 2. Install Certificates on NAS
Back on your terminal (SSH):
```bash
# Switch to root
sudo -i

# Create directory
mkdir -p /data/coolify/proxy/certs
cd /data/coolify/proxy/certs

# Paste Cert
nano domain.cert
# (Paste -> Ctrl+X -> Y)

# Paste Key
nano domain.key
# (Paste -> Ctrl+X -> Y)

# Exit root
exit
```

### 3. Configure Coolify Proxy
1.  In Coolify, go to **Servers** -> `localhost` -> **Proxy** tab -> **Dynamic Configuration**.
2.  Click **Add**.
3.  **Name:** `cloudflare-origin`
4.  **Configuration (YAML):**
    ```yaml
    tls:
      certificates:
        - certFile: /traefik/certs/domain.cert
          keyFile: /traefik/certs/domain.key
    ```
5.  Click **Save**.

### 4. Switch Tunnel to HTTPS (Cloudflare Dashboard)
Now we tell the tunnel to expect HTTPS.
1.  Go to **Zero Trust** -> **Networks** -> **Tunnels** -> **beelink-nas** -> **Configure**.
2.  **Public Hostname** tab -> **Add Public Hostname**:
    *   **Subdomain:** `coolify` -> **Domain:** `yourdomain.com`.
    *   **Service:** `HTTPS` -> `beelink-nas.your-tailnet.ts.net:443`
        *   *Note: Using your Tailscale MagicDNS name here is more stable than `localhost`.*
        *   *If unsure, check Phase 2 for your specific MagicDNS name.*
    *   **Additional Settings:**
        *   **No TLS Verify:** `Disabled` (We want verification).
        *   **Origin Server Name:** `yourdomain.com` (Matches your cert).
3.  **Wildcard Mapping:**
    *   **Subdomain:** `*` -> **Domain:** `yourdomain.com`.
    *   **Service:** `HTTPS` -> `beelink-nas.your-tailnet.ts.net:443`
    *   **Additional Settings:**
        *   **Origin Server Name:** `yourdomain.com`
4.  **SSL Mode:** Go to Cloudflare **SSL/TLS** -> **Overview** -> Set **Full (Strict)**.

### 5. Update Coolify URLs (Crucial)
1.  **Instance Domain:** Settings -> Instance's Domain -> Change `http://` to `https://`.
    ![Instance Settings](/C:/Users/josep/.gemini/antigravity/brain/16603f28-23c4-4583-a96d-1732550059e5/uploaded_image_1_1764991205365.png)
2.  **Wildcard Domain (The Magic Step):**
    *   Go to **Servers** -> **localhost** -> **General** tab.
    *   Find the field **Wildcard Domain**.
    *   Enter your root domain with `https://` (e.g., `https://yourdomain.com`).
    *   **CRITICAL:** Do NOT put an asterisk (`*`) here. Just the domain.
    *   Click **Save**.
3.  **Resources:** For future apps, they will now automatically get `https://app.yourdomain.com`.

---

## 🛡️ Phase 6: Security Hardening (Ubuntu)
*Goal: Basic security hygiene to prevent unauthorized access.*

Now that everything is working, we lock the door behind us.

### 1. Disable Password Authentication
Since we have Tailscale for identity (or SSH keys), we should turn off password logins to prevent brute-force attacks.

```bash
# Edit the SSH config
sudo nano /etc/ssh/sshd_config
```

1.  Find the line `#PasswordAuthentication yes` (Use `Ctrl+W` to search).
2.  Change it to: `PasswordAuthentication no`.
3.  Remove the `#` if it exists.
4.  Save and Exit (`Ctrl+X`, then `Y`, then `Enter`).

### 2. Restart SSH
Apply the changes immediately.

```bash
sudo systemctl restart ssh
```

> [!CAUTION]
> **Verify Access:** Before closing your current terminal window, open a **new** terminal window and try to SSH in again. If it works, you are safe. If not, you still have your current window open to fix `sshd_config`.

---

## 🛠️ Phase 7: Service Launch (IT Tools)
*Goal: Install our first app to learn about "Environments" and good practices.*

We will install **IT Tools**, a handy utility suite (token generators, code formatters, etc.).

1.  **Open your Project:**
    *   Go to **Projects** -> **"System and Admin Tools"**.
2.  **Create a New Environment:**
    *   You will see the default "Production" environment (where our Tunnel lives).
    *   Environments help organize stages (e.g., Dev, Staging, Prod). We will create a new one for tools.
    *   Click **+ Add Environment**.
    *   Name it: `Admin Tools`.
    *   Open the new **Admin Tools** environment.
3.  **Configure & Deploy IT Tools:**
    *   Click **+ New** -> **Resource** -> Search for **"IT Tools"**.
    *   **Customize Domain:**
        *   You'll see a long, random domain (e.g., `https://it-tools-t084...`). Let's fix that.
        *   In the **General** settings tab, locate the **Domains** box.
        *   Change the domain to: `https://tools.yourdomain.com:80`.
        *   **CRITICAL:** Ensure you use `https://` protocol and add `:80` at the end.
        *   Click **Save**.
    *   **Deploy:**
        *   Now click **Deploy** (top right).
4.  **Verify:**
    *   Look for the **Links** button (top right of the Coolify dashboard).
    *   Click it to see a clickable dropdown of your domains.
    *   Select your link (e.g., `https://tools.yourdomain.com`).
    *   Use the **Token Generator** to make a strong password for your next service!

### 5. Secure with Cloudflare Zero Trust (OTP)
*Goal: Add a "PIN Code" (One-Time Password) so only you can access IT Tools.*

Since IT Tools is powerful, we don't want the public to see it. We will use Cloudflare Access to protect it.

1.  Go to **Cloudflare Dashboard** -> **Zero Trust**.
2.  Navigate to **Access** -> **Applications** -> **Add an Application**.
3.  Select **Self-hosted**.
4.  **Configure:**
    *   **Application Name:** `IT Tools`
    *   **Session Duration:** `24 hours` (or your preference).
    *   **Subdomain:** `tools` -> **Domain:** `yourdomain.com` (Matches what we set in Coolify).
5.  **Policy (The PIN):**
    *   **Policy Name:** `Allow Admin`
    *   **Action:** `Allow`
    *   **Configure Rules:**
        *   **Include:** Select `Emails` -> Enter your email address.
    *   *Note: This will send a 6-digit PIN to your email when you try to access the site.*
6.  **Save Application.**
    *   Now, when you visit `https://tools.yourdomain.com`, you will be asked for a code. Secure!

---

---

## 🔐 Phase 8: Service Launch (Vault Warden)
*Goal: Self-hosted password management (Bitwarden compatible).*

> [!CRITICAL]
> **Security Check:** Before proceeding, ensure you have followed **all** security best practices in this guide.
> 1.  **Strong Passwords:** Did you use strong, generated passwords for Ubuntu (Phase 1) and Coolify?
> 2.  **Hardened SSH:** Did you disable password authentication (Phase 6)?
>
> If you answered "No" to either, **STOP**. Do not install a password manager until your server foundation is secure.

*(Detailed installation steps to follow. For now, prioritize verifying your security foundation.)*

---

## 🚀 Phase 9: Advanced Service Launch (n8n)
*Goal: Deploy a complex automation platform.*

1.  In Coolify, create a **New Project** named **"Automation"**.
    *   *Alternatively, you can keep organizing in "System and Admin Tools" if you prefer, but a new project is cleaner.*
2.  Select **"Production"** environment.
3.  Add Resource -> **n8n**.
4.  **Domain:** `https://automation.yourdomain.com`.
    > [!IMPORTANT]
    > **Why 'automation' and not 'n8n'?**
    > Google Chrome often falsely flags `n8n.yourdomain.com` as a "Deceptive Site" (Phishing) because the login page looks like the official cloud login. Using a generic name like `automation` or `flow` usually prevents this warning.
5.  **Deploy!**
    *   Coolify routes it via Traefik (using your Origin Cert).
    *   Cloudflare Tunnel sends it encrypted to Traefik.
    *   Secure. Accessible. Done.
