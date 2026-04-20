# Windows Server 2022 + Active Directory Setup Guide
### Using Oracle VirtualBox + Cloudflare Tunnel for Remote Access

---

## 📋 Prerequisites

- Oracle VirtualBox installed on your host PC
- Windows Server 2022 Evaluation ISO (downloaded from [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022))
- At least 4GB RAM and 50GB free disk space on host PC
- Stable internet connection
- Free Cloudflare account

---

## PART 1 — Create the Virtual Machine

1. Open **VirtualBox** → Click **New**
2. Fill in:
   - **Name:** `WinServer2022`
   - **ISO Image:** Browse to your downloaded `.iso` file
   - **Type:** Microsoft Windows
   - **Version:** Windows Server 2022 (64-bit)
   - ⚠️ **Uncheck** "Proceed with Unattended Installation"
3. Click **Next**
4. Set hardware:
   - **Base Memory:** 4096 MB
   - **CPUs:** 2
5. Click **Next**
6. Set disk:
   - **Disk Size:** 50 GB
7. Click **Next** → **Finish**

### Configure Network (Before Starting)
1. Select your VM → click **Settings**
2. Go to **Network** → **Adapter 1**
3. Change **NAT** to → **Bridged Adapter**
4. Under **Name**, select your active Wi-Fi or Ethernet card
5. Click **OK**

---

## PART 2 — Install Windows Server 2022

1. Select your VM → click **Start**
2. When the setup screen appears:
   - **Language:** English (United States)
   - **Time:** your local timezone
   - **Keyboard:** US
   - Click **Next** → **Install Now**
3. When asked for a product key → click **"I don't have a product key"**
4. Choose edition: **Windows Server 2022 Standard Evaluation (Desktop Experience)**
5. Accept license → click **Next**
6. Select **"Custom: Install Windows only (advanced)"**
7. Select **Drive 0 Unallocated Space** → click **Next**
8. Wait 15–25 minutes for installation to complete (VM restarts automatically)

### First Boot — Set Administrator Password
- When prompted, set a strong password that includes:
  - Uppercase letters
  - Lowercase letters
  - Numbers
  - Symbols
  - Example format: `MyPassword@2024!`
- Write this password down safely

### Set Password via PowerShell (if blank password was set)
Open PowerShell as Administrator and run:
```powershell
net user Administrator YourStrongPassword@2024!
```

---

## PART 3 — Install VirtualBox Guest Additions

1. In the VirtualBox menu bar → click **Devices**
2. Click **"Insert Guest Additions CD image..."**
3. Inside the VM, open **File Explorer** → go to **CD Drive (D:)**
4. Double-click **VBoxWindowsAdditions-amd64**
5. Click **Next** → **Next** → **Install** → **Reboot Now**

---

## PART 4 — Install Active Directory (AD DS)

1. Open **Server Manager** → click **"Add roles and features"**
2. Follow the wizard:
   - **Installation Type:** Role-based or feature-based installation → Next
   - **Server Selection:** your server is already selected → Next
   - **Server Roles:** Check **"Active Directory Domain Services"** → click **"Add Features"** → Next
   - **Features:** Leave defaults → Next
   - **AD DS:** Next
   - **Confirmation:** Click **Install**
3. Wait 3–5 minutes for installation

### Promote to Domain Controller
1. After install, click the **yellow flag ⚑** in Server Manager
2. Click **"Promote this server to a domain controller"**
3. Fill in the wizard:
   - **Deployment:** Select **"Add a new forest"**
   - **Root domain name:** `mylab.local`
   - Click **Next**
4. **Domain Controller Options:**
   - Leave forest/domain functional level as **Windows Server 2016**
   - Leave DNS Server and Global Catalog checked
   - Set a **DSRM password** (write it down!)
   - Click **Next**
5. **DNS Options:** Ignore the warning → click **Next**
6. **Additional Options:** Wait for `MYLAB` to auto-fill → click **Next**
7. **Paths:** Leave defaults → click **Next**
8. **Review Options:** click **Next**
9. **Prerequisites Check:** Click **Install**
10. Server will automatically restart — Active Directory is now live!

---

## PART 5 — Configure Network Settings

### Set Static IP via PowerShell
Open PowerShell as Administrator inside the VM:

```powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.200 -PrefixLength 24 -DefaultGateway 192.168.1.1

# Set DNS (server points to itself for AD)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1,8.8.8.8
```

> ⚠️ Check your actual gateway first with `ipconfig` and adjust accordingly

---

## PART 6 — Enable Remote Desktop

Run these commands in PowerShell as Administrator:

```powershell
# Enable RDP
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

# Allow RDP through firewall
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Disable NLA requirement (for easier access)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0
```

---

## PART 7 — Set Up Cloudflare Tunnel (Remote Access)

This allows you to access your server from **anywhere** securely.

### Step 1 — Create a Cloudflare Account
- Sign up at [cloudflare.com](https://cloudflare.com) (free)

### Step 2 — Create a Tunnel
1. Go to Cloudflare Dashboard → **Networking** → **Tunnels**
2. Click **"Create a Tunnel"**
3. Name it: `winserver-rdp`
4. Select **Windows** + **64-bit**
5. Copy the install command shown (keep it secret!)

### Step 3 — Download cloudflared on the VM
Open PowerShell as Administrator in the VM:

```powershell
# Download cloudflared installer
Invoke-WebRequest -Uri "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.msi" -OutFile "C:\cloudflared.msi"

# Install cloudflared
Start-Process msiexec.exe -Wait -ArgumentList '/I C:\cloudflared.msi /quiet'
```

### Step 4 — Install the Tunnel Service
In PowerShell, type this command and paste your token at the end:

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" service install YOUR_TUNNEL_TOKEN
```

> ⚠️ Never share your tunnel token publicly — treat it like a password!

### Step 5 — Start the Service
```powershell
Start-Service cloudflared
Get-Service cloudflared  # Should show "Running"
```

### Step 6 — Add a Route in Cloudflare Dashboard
1. Go to your tunnel → click **Routes** tab
2. Click **"+ Add route"**
3. Select **"Private CIDR"**
4. Enter: `192.168.1.200/32`
5. Click **Save**

---

## PART 8 — Create Active Directory Users

1. Open **Server Manager** → **Tools** → **Active Directory Users and Computers**
2. Expand `mylab.local` → click **Users**
3. Right-click → **New** → **User**
4. Fill in user details → set a password
5. After creating the user, add them to **Remote Desktop Users** group:

```powershell
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "MYLAB\username"
```

---

## PART 9 — Connect via Remote Desktop

### Same Network (Local)
On any device on the same Wi-Fi:
1. Open **Remote Desktop Connection** (`mstsc`)
2. Enter: `192.168.1.200`
3. Login with:
   - **Username:** `MYLAB\Administrator` or `MYLAB\yourusername`
   - **Password:** your password

### Remote Access (Outside Network)
1. Install **Cloudflare WARP** on your device from [one.one.one.one](https://one.one.one.one)
2. Connect WARP to your Cloudflare Zero Trust account
3. Then RDP to `192.168.1.200` as normal

---

## 📝 Important Credentials to Save

| Item | Value |
|------|-------|
| **Domain** | mylab.local |
| **Server IP** | 192.168.1.200 |
| **Admin Username** | MYLAB\Administrator |
| **Admin Password** | (your password) |
| **DSRM Password** | (your DSRM password) |

---

## 🔒 Security Notes

- Never expose RDP (port 3389) directly to the internet
- Always use Cloudflare Tunnel or VPN for remote access
- Keep Windows Server updated regularly
- Use strong passwords for all accounts
- Never share tunnel tokens or credentials publicly

---

## ✅ Final Architecture

```
Your Browser / RDP Client
         │
         ▼
Cloudflare WARP (secure tunnel)
         │
         ▼
Cloudflare Tunnel (winserver-rdp)
         │
         ▼
Windows Server 2022 VM (192.168.1.200)
         │
         └── Active Directory (mylab.local)
                  └── Users & Groups
```

---

*Built with Oracle VirtualBox + Windows Server 2022 Standard Evaluation + Cloudflare Tunnel*
