### Step 1: Hardware & Software Requirements

Before starting the setup, ensure you have the following hardware and software ready. This setup is designed for a high-performance, isolated Diretta audio chain.

---

#### Hardware

* LHY OCK-2S  
* LHY Audio AS8 Pro  
* LHY FMC  
* LHY RPI  
* LHY RPI Pro  
* DAC  
* Music server (PC / laptop)

---

#### Software 

* AudioLinux license (paid)  
  * Website: [AudioLinux official website](https://www.audio-linux.com/)  
  * Cost: typically around €49  

* Diretta license  
  * Website: [Diretta official website](https://www.diretta.link/)  
  * Cost: approximately €100  

---

#### 1.1 System Architecture Overview

```
Music Server (PC / Laptop)
        │
     Ethernet
        │
LHY AS8 Pro (clocked by OCK-2S)
        │
   SFP / Ethernet
        │
   LHY FMC (Fiber Isolation)
        │
     Ethernet
        │
   LHY RPI Pro (Diretta Host)
        │
        │  Direct Ethernet (Isolated Link)
        │
   LHY RPI (Diretta Target)
        │
        │  USB
        │
        ↓
        DAC
```

---
### Step 2: Flashing AudioLinux on SD Cards

**Download AudioLinux**
* Download the AudioLinux image from the [AudioLinux official website](https://www.audio-linux.com/). You will receive a link to download a `.img.gz` or `.img.xz` file via email typically within 24 hours of purchase.

**Flash 2x SD cards with the image you just downloaded**
* Open Raspberry Pi Imager
  - Select: “Use custom image"
* Choose the downloaded
  - .img.gz or .img.xz file (no need to extract)
* Select your microSD card
* Click Write

**Important Notes**
* Flash both SD cards separately using the same image
* Do not modify files manually after flashing

**Alternative Flashing Tools (Optional)**
* Windows: balenaEtcher, Win32 Disk Imager
* macOS: balenaEtcher, ApplePi-Baker
* Linux: balenaEtcher, dd command-line tool

---

### Step 3: Configuring Diretta HOST

We shall configure the Diretta HOST first and then move on to the Diretta TARGET.

Make sure only the HOST device is turned ON during this step, turn the TARGET device OFF.

Refer to the E-mail from your Audiolinux purchase for the default SSH user and sudo/root passwords.

Obtain the IP of your HOST from your Router's App or Web UI.

SSH into the Diretta HOST using this command.

```bash
ssh audiolinux@<HOST_IP_ADDRESS>
```

Regenerate Machine ID for the HOST machine

```bash
echo ""
echo "Old Machine ID: $(cat /etc/machine-id)"
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
echo "New Machine ID: $(cat /etc/machine-id)"
```

Set Hostname

```bash
sudo hostnamectl set-hostname diretta-host
```

Shutdown

```bash
sudo sync && sudo poweroff
```

---

### Step 4: Configuring Diretta TARGET

We shall configure the Diretta TARGET now.

Make sure only the TARGET device is turned ON during this step, turn the HOST device OFF.

Refer to the E-mail from your Audiolinux purchase for the default SSH user and sudo/root passwords.

Obtain the IP of your TARGET from your Router's App or Web UI.

SSH into the Diretta TARGET using this command.

```bash
ssh audiolinux@<TARGET_IP_ADDRESS>
```

Regenerate Machine ID for the TARGET machine

```bash
echo ""
echo "Old Machine ID: $(cat /etc/machine-id)"
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
echo "New Machine ID: $(cat /etc/machine-id)"
```

Set Hostname

```bash
# On the Diretta Target
sudo hostnamectl set-hostname diretta-target
```

Shutdown

```bash
sudo sync && sudo poweroff
```

---

### Step 5: System Stability

This step prepares the system for stable operation, ensures correct time and networking, updates AudioLinux, and sets a compatible kernel.

Run the following commands:

```bash
sudo id
curl -fsSL https://raw.githubusercontent.com/dsnyder0pc/rpi-for-roon/refs/heads/main/scripts/setup_chrony.sh | sudo bash
sleep 5
chronyc sources
```

Set your timezone:

```bash
cmd=$(cat <<'EOT'
clear
echo "Welcome to the interactive timezone setup."
echo "You will first select a region, then a specific timezone."

PS3="
Please select a number for your region: "
select region in $(timedatectl list-timezones | grep -F / | cut -d/ -f1 | sort -u); do
  if [[ -n "$region" ]]; then
    echo "You have selected the region: $region"
    break
  else
    echo "Invalid selection. Please try again."
  fi
done

echo ""

PS3="
Please select a number for your timezone: "
select timezone in $(timedatectl list-timezones | grep "^$region/"); do
  if [[ -n "$timezone" ]]; then
    echo "You have selected the timezone: $timezone"
    break
  else
    echo "Invalid selection. Please try again."
  fi
done

echo
echo "Setting timezone to ${timezone}..."
sudo timedatectl set-timezone "$timezone"

echo
echo "Current system time and timezone:"
timedatectl status
EOT
)
bash -c "$cmd"
```

Install DNS utilities required for updates:

```bash
sudo pacman -S --noconfirm --needed dnsutils
```

Update the system using the AudioLinux menu:

* Run `menu`
* Go to INSTALL/UPDATE
* Select UPDATE system and wait for completion
* Then select Update menu
* Exit back to terminal

Select the correct kernel:

* Run `menu`
* Go to Update/Install
* Select Kernel update
* Choose: Audiolinux LTS RT LTO (6.12.59-1)

If the system update fails due to firmware conflict, run:

```bash
sudo pacman -Rdd --noconfirm linux-firmware
sudo pacman -Syu --noconfirm linux-firmware
```

Reboot to apply all changes:

```bash
sudo sync && sudo reboot
```

---
### Step 6: Point-to-Point Network Configuration & Isolation

In this step, a **dedicated, isolated network link** is configured between the Diretta Host and Target, while integrating an upstream **clocked and fiber-isolated network chain**.

This approach ensures:

* minimal electrical noise
* stable timing
* maximum audio performance

Perform all steps while both devices are still connected to the main LAN via SSH.

---

#### 6.1 Why Routed Network Instead of Bridge

A bridge configuration is intentionally avoided:

* A bridge exposes the Target to LAN broadcast and multicast traffic
* This increases CPU activity and electrical interference
* A routed + NAT setup creates a **fully isolated subnet**
* The Host operates as a **firewall and controller**, maintaining a minimal processing load on the Target

This design provides an optimal environment for Diretta operation.

---

#### 6.2 Pre-configure the Diretta Host

Create Network Configuration Files

```bash
cat <<'EOT' | sudo tee /etc/systemd/network/end0.network
[Match]
Name=end0

[Link]
MTUBytes=1500

[Network]
Address=172.20.0.1/24
EOT
```

```bash
cat <<'EOT' | sudo tee /etc/systemd/network/usb-uplink.network
[Match]
Name=en[pu]*

[Link]
MTUBytes=1500

[Network]
DHCP=yes
DNSSEC=no
EOT
```

Remove Conflicting Network Files

```bash
sudo rm -fv /etc/systemd/network/{en,enp,auto,eth}.network
```

Add Host Entry for Target

```bash
HOSTS_FILE="/etc/hosts"
TARGET_IP="172.20.0.2"
TARGET_HOST="diretta-target"

if ! grep -q "$TARGET_IP\s\+$TARGET_HOST" "$HOSTS_FILE"; then
  printf "%s\t%s target\n" "$TARGET_IP" "$TARGET_HOST" | sudo tee -a "$HOSTS_FILE"
fi
```

Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forwarding.conf
```

Configure NAT using nftables

```bash
sudo pacman -S --noconfirm --needed nftables

cat <<'EOT' | sudo tee /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table ip my_table {

    chain prerouting {
        type nat hook prerouting priority dstnat;
        tcp dport 5101 dnat to 172.20.0.2:5001
    }

    chain forward {
        type filter hook forward priority 0;
        policy drop;

        ct state established,related accept
        ip daddr 172.20.0.2 tcp dport 5001 ct state new accept
        ip saddr 172.20.0.0/24 accept
    }

    chain postrouting {
        type nat hook postrouting priority 100;

        ip saddr 172.20.0.0/24 oifname "enp*" masquerade
        ip saddr 172.20.0.0/24 oifname "enu*" masquerade
        ip saddr 172.20.0.0/24 oifname "wlp*" masquerade
    }
}
EOT

sudo systemctl disable --now iptables.service 2>/dev/null
sudo rm /etc/iptables/iptables.rules 2>/dev/null
sudo systemctl enable --now nftables.service
```

Fix USB Ethernet Adapter

```bash
cat <<'EOT' | sudo tee /etc/udev/rules.d/99-ax88179a.rules
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="0b95", ATTR{idProduct}=="1790", ATTR{bConfigurationValue}!="1", ATTR{bConfigurationValue}="1"
EOT

sudo udevadm control --reload-rules
```

Fix MOTD Script

```bash
[ -f /opt/scripts/update/update_motd.sh.dist ] || \
sudo mv /opt/scripts/update/update_motd.sh /opt/scripts/update/update_motd.sh.dist

curl -LO https://raw.githubusercontent.com/dsnyder0pc/rpi-for-roon/refs/heads/main/scripts/update_motd.sh
sudo install -m 0755 update_motd.sh /opt/scripts/update/
rm update_motd.sh
```

Shutdown Host

```bash
sudo sync && sudo poweroff
```

---

#### 6.3 Pre-configure the Diretta Target

Create Network File

```bash
cat <<'EOT' | sudo tee /etc/systemd/network/end0.network
[Match]
Name=end0

[Link]
MTUBytes=1500

[Network]
Address=172.20.0.2/24
Gateway=172.20.0.1
DNS=1.1.1.1
EOT
```

Remove Conflicting Files

```bash
sudo rm -fv /etc/systemd/network/{en,auto,eth}.network
```

Add Host Entry for Host

```bash
HOSTS_FILE="/etc/hosts"
HOST_IP="172.20.0.1"
HOST_NAME="diretta-host"

if ! grep -q "$HOST_IP\s\+$HOST_NAME" "$HOSTS_FILE"; then
  printf "%s\t%s host\n" "$HOST_IP" "$HOST_NAME" | sudo tee -a "$HOSTS_FILE"
fi
```

---

#### 6.4 Physical Network Setup

Double-check all configurations before proceeding.

Shutdown both devices

```bash
sudo sync && sudo poweroff
```

Network Topology

* Music Server (Roon Core / HQPlayer / PC) → LHY AS8 Pro
* LHY AS8 Pro (clocked by OCK-2S) → LHY FMC (Fiber Media Converter)
* LHY FMC → Diretta Host via Ethernet
* Diretta Host → Diretta Target via **direct Ethernet link**
* Diretta Target → DAC via USB

Topology Diagram

```
Music Server (Roon / HQPlayer / PC)
        │
     Ethernet
        │
LHY AS8 Pro (clocked by OCK-2S)
        │
   SFP / Ethernet
        │
   LHY FMC (Fiber Isolation)
        │
     Ethernet
        │
   Diretta Host
        │
        │  Direct Ethernet (172.20.0.x isolated link)
        │
   Diretta Target
        │
        │
        │  USB
        │
        ↓
        DAC
```

Key Design Notes

* The Host–Target connection remains **direct and isolated**
* No network switch is present between Host and Target
* Upstream network components (switch, clock, fiber) are placed **before the Host**
* The Host functions as:

  * a network boundary
  * a traffic filter
  * the Diretta control node

This architecture combines:

* upstream signal conditioning
* downstream isolation

Power On Sequence

* Power on Target first
* Power on Host afterward

---

#### 6.5 Verification

SSH into Host

```bash
ssh audiolinux@<HOST_IP>
```

Test Target Connectivity

```bash
ping -c 3 172.20.0.2
```

SSH into Target

```bash
ssh target
```

Test Internet from Target

```bash
ping -c 3 one.one.one.one
```

---
### Step 7: Diretta Software Installation & Configuration

#### 7.1 On the Diretta Target

1. Connect your USB DAC to one of the black USB 2.0 ports on the **Diretta Target** and ensure the DAC is powered on.
2. SSH to the Target: `ssh diretta-target`.
3. Configure Compatible Compiler Toolchain

```bash
curl -fsSL https://raw.githubusercontent.com/dsnyder0pc/rpi-for-roon/refs/heads/main/scripts/setup_diretta_compiler.sh | sudo bash
```

4. Run `menu`.

5. Select **AUDIO extra menu**.

6. Select **DIRETTA target installation/configuration**. You will see the following menu:

```text
What do you want to do?

0) Install previous stable version
1) Install/update last version
2) Enable/Disable Diretta Target
3) Configure Audio card
4) Edit configuration
5) Copy and edit new default configuration
6) License
7) Diretta Target log
8) Exit

?
```

7. You should perform these actions in sequence:

* Choose **1) Install/update** to install the software (say "Y" to all prompts.)
* Choose **2) Enable/Disable Diretta Target** and enable it.
* Choose **3) Configure Audio card**. The system will list your available audio devices. Enter the card number corresponding to your USB DAC.

```text
?3
This option will set DIRETTA target to use a specific card
Your available cards are:

card 0: AUDIO [SMSL USB AUDIO], device 0: USB Audio [USB Audio]

Please type the card number (0,1,2...) you want to use:
?0
```

* Choose **4) Edit configuration**. Set `AlsaLatency=20` for a Raspberry Pi 5 Target or `AlsaLatency=40` for RPi4.
* Choose **6) License**. The system will play hi-res (greater than 44.1 kHz PCM audio) for 6 minutes in trial mode. Follow the on-screen link and instructions to purchase and apply your full license for hi-res support. This requires the internet access configured earlier.

```text
The price of this third party license is 100$
Without license DIRETTA Target will work for 6 min.
If you see a link, you can use it to purchase a license
If you see instead the word 'valid' the license has been correctly applied
Please wait a few seconds...

https://www.diretta.link/cgi-bin/target_app_regist.cgi?hash=1fd430fe950936867b31cc084a9dac031ffa7c57c8ba1d5034a1a5219444f441&vender=Audlinux


The license will be applied at the next DIRETTA target start
Press any key to continue
```

* Choose **8) Exit**. Follow prompts to get back to the terminal

---

#### 7.2 On the Diretta Host

1. SSH to the Host: `ssh diretta-host`.

2. Configure Compatible Compiler Toolchain

```bash
curl -fsSL https://raw.githubusercontent.com/dsnyder0pc/rpi-for-roon/refs/heads/main/scripts/setup_diretta_compiler.sh | sudo bash
```

3. Run `menu`.

4. Select **AUDIO extra menu**.

5. Select **DIRETTA host installation/configuration**. You will see the following menu:

```text
What do you want to do?

0) Install previous stable version
1) Install/update last version
2) Enable/Disable Diretta daemon
3) Set Ethernet interface
4) Edit configuration
5) Copy and edit new default configuration
6) Diretta log
7) Exit

?
```

6. You should perform these actions in sequence:

* Choose **1) Install/update** to install the software. (say "Y" to all prompts.) *(Note: you may see `error: package 'lld' was not found. Don't worry, that will be corrected automatically by the installation)*

* Choose **2) Enable/Disable Diretta daemon** and enable it.

* Choose **3) Set Ethernet interface**. It is critical to select `end0`, the interface for the point-to-point link.

```text
?3
Your available Ethernet interfaces are: end0  enu1
Please type the name of your preferred interface:
end0
```

* Choose **4) Edit configuration** only if you need to make advanced changes. The previous steps should be sufficient; however, here are some tuned settings you may wish to try:

```text
FlexCycle=disable
CycleTime=800
periodMin=16
periodSizeMin=2048
```

* If you just want to install the tuned parameters above, you can use this command block:

```bash
cat <<'EOT' | sudo tee /opt/diretta-alsa/setting.inf
[global]
Interface=end0
TargetProfileLimitTime=200
ThredMode=1
InfoCycle=100000
FlexCycle=disable
CycleTime=800
CycleMinTime=
Debug=stdout
periodMax=32
periodMin=16
periodSizeMax=38400
periodSizeMin=2048
syncBufferCount=8
alsaUnderrun=enable
unInitMemDet=disable
CpuSend=
CpuOther=
LatencyBuffer=0
disConnectDelay=enable
singleMode=
EOT
```

* Choose **7) Exit**. Follow prompts to get back to the terminal

7. Create an override to make the Diretta service auto-restart on failure

```bash
sudo mkdir -p /etc/systemd/system/diretta_alsa.service.d
cat <<'EOT' | sudo tee /etc/systemd/system/diretta_alsa.service.d/restart.conf
[Service]
Restart=on-failure
RestartSec=5
EOT
```

---

#### Completion

At this stage:

* Diretta Target is configured and linked to the DAC
* Diretta Host is configured and bound to the isolated network interface
* Communication between Host and Target is fully established

---
### Step 8: Final Steps & Roon Integration

Run menu if you exited back to the terminal after the previous step, otherwise go to the Main menu.

Install Roon Bridge (on Host): If you use Roon, perform the following steps on the Diretta Host:

Run menu.

Select INSTALL/UPDATE menu.

Select INSTALL/UPDATE Roonbridge.

The installation will proceed. The installation may take a minute or two.

Enable Roon Bridge (on the Host):

Select Audio menu from the Main menu

Select SHOW audio service

If you don't see roonbridge under enabled services, select ROONBRIDGE enable/disable

Reboot Both Devices: For a clean start, reboot both the Target and Host, in that order:

```bash
sudo sync && sudo reboot
```

Configure Roon:

Open Roon on your control device.

Go to Settings -> Audio.

Under diretta-host, you should see your device. The name will be based on your DAC.

Click Enable, give it a name, and you are ready to play music!

Your dedicated Diretta link is now fully configured for pristine, isolated audio playback.

> Note: The "Limited" zone for Diretta Target testing will disappear from Roon after six minutes of hi-res music playback. This is normal. At that point, a license for the Diretta Target is required. Cost is currently €100 and activation can take up to 48 hours. Two emails will be received from the Diretta team: the first is the receipt, and the second confirms activation. Once the activation email is received, restart the Target to apply the license.

---
