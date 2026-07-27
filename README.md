# Sleep and Wake a Remote Server with Wake-On-LAN

Two systemd services that keep a home server in sync with this machine's power state: one shuts the server down over SSH whenever the PC shuts down, the other wakes it via Wake-on-LAN whenever the PC boots.

Is WOL supported? Go to step 6 to check FIRST!

I personally use a HP Mini PC as homeserver, for the very reason that it supports WOL by default!.

## Why not just use `/etc/systemd/system-shutdown/`?

Scripts placed in `/etc/systemd/system-shutdown/` run *very* late in the shutdown sequence - after systemd has already torn down networking and most other services. Any script there that needs the network (like an SSH call) will silently fail with a connection timeout.

The fix is to define a proper systemd **oneshot service** whose `ExecStop=` runs earlier, while the network is still up.

## Setup

### 1. Make sure you have a working SSH key

You need a **passphrase-less** SSH key, because systemd runs services non-interactively (no terminal, no SSH agent) - it can't prompt for a passphrase.

Check for an existing key:
```bash
ls -la ~/.ssh/
```

If you don't have a passphrase-less key, generate a dedicated one just for this purpose:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/shutdown_key -N ""
```

### 2. Copy the public key to the remote server

```bash
ssh-copy-id -i ~/.ssh/shutdown_key.pub server@192.168.1.10
```
(or manually append the `.pub` file's contents to `~/.ssh/authorized_keys` on the remote server)

Test it logs in without a prompt:
```bash
ssh -i ~/.ssh/shutdown_key server@192.168.1.10 "echo it works"
```

### 3. Allow passwordless `sudo shutdown` on the remote server

On the **remote** server, edit sudoers (`sudo visudo`) and add:
```
server ALL=(ALL) NOPASSWD: /usr/sbin/shutdown
```
(replace `server` with the actual remote username)

### 4. Create the systemd service on this machine

```bash
sudo tee /etc/systemd/system/notify-server-shutdown.service > /dev/null << 'EOF'
[Unit]
Description=Notify remote server on shutdown
Requires=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/true
ExecStop=/bin/bash -c '/usr/bin/ssh -i /home/YOUR_USERNAME/.ssh/shutdown_key -o ConnectTimeout=5 -o StrictHostKeyChecking=no server@192.168.1.10 "sudo /usr/sbin/shutdown now" >> /var/log/sleep-server.log 2>&1'

[Install]
WantedBy=multi-user.target
EOF
```

Replace `YOUR_USERNAME`, the key path, and the remote user/IP with your own values.

**Notes on why it's built this way:**
- `ExecStart=/bin/true` does nothing - it just puts the service in an "active" state at boot.
- `ExecStop=` is what actually runs, and it fires when the service is stopped - which happens automatically during normal shutdown, while the network is still up.
- The command is wrapped in `bash -c '...'` because systemd does **not** use a shell to run `ExecStart`/`ExecStop` - without it, `>>` and `2>&1` are passed as literal arguments instead of being interpreted as redirection.
- Avoid `DefaultDependencies=no` combined with `Before=network.target` - it creates an ordering cycle. `After=network-online.target` alone is enough, since systemd stops services in reverse order of starting.
- The log is written to `/var/log/` rather than `/tmp/` to avoid permission issues (root owns the service, and `/tmp` files from earlier manual tests can end up owned by your user).

### 5. Enable and reload

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now notify-server-shutdown.service
```

## Waking the server back up

Putting the server to sleep is only half the story - you'll also want a way to wake it back up with Wake-on-LAN (WOL).

### 6. Find the server's Ethernet adapter and MAC address

Run these **on the remote server**, not on this machine.

List network interfaces to find the Ethernet adapter name (usually something like `eth0`, `enp3s0`, or `eno1` - Wi-Fi adapters won't support WOL):
```bash
ip link show
```

Get the MAC address for that adapter:
```bash
ip link show enp3s0
```
Look for the `link/ether` line, e.g. `link/ether aa:bb:cc:dd:ee:ff`.

Check whether WOL is currently supported/enabled on that adapter (requires `ethtool`, `sudo apt install ethtool` if missing):
```bash
sudo ethtool enp3s0 | grep Wake-on
```
- `Supports Wake-on: pumbg` - the adapter supports it
- `Wake-on: d` - currently **disabled** (`d` = disabled)
- `Wake-on: g` - currently **enabled** (`g` = magic packet)

Enable it if needed:
```bash
sudo ethtool -s enp3s0 wol g
```

This setting is often lost on reboot, so make it persistent - for example with a `systemd-networkd` drop-in, a NetworkManager dispatcher script, or a small systemd service depending on your distro. Check your distro's docs for the cleanest way to persist `ethtool -s <iface> wol g` at boot.

### 7. Install `wakeonlan` on this machine

`wakeonlan` sends the magic packet that wakes the server. Install it on **this** machine (the one issuing the wake command), not the server:

```bash
sudo apt install wakeonlan
```

Test it manually:
```bash
wakeonlan aa:bb:cc:dd:ee:ff
```
(replace with the server's actual MAC address from step 6)

## The two scripts

Two standalone scripts, kept separate so each can be run independently - e.g. wake the server manually before starting a work session, or wire `sleep-server.sh` into a cron job, hotkey, or the systemd service above.

### `sleep-server.sh`

```bash
#!/bin/bash
# Puts the remote server to sleep/shutdown over SSH.

SERVER_USER="server"
SERVER_IP="192.168.1.10"
SSH_KEY="/home/YOUR_USERNAME/.ssh/shutdown_key"
LOG="/var/log/sleep-server.log"

/usr/bin/ssh -i "$SSH_KEY" -o ConnectTimeout=5 -o StrictHostKeyChecking=no \
  "$SERVER_USER@$SERVER_IP" "sudo /usr/sbin/shutdown now" \
  >> "$LOG" 2>&1
```

### `wake-server.sh`

```bash
#!/bin/bash
# Wakes the remote server via Wake-on-LAN.

SERVER_MAC="aa:bb:cc:dd:ee:ff"
LOG="/var/log/wake-server.log"

/usr/bin/wakeonlan "$SERVER_MAC" >> "$LOG" 2>&1
```

Make both executable:
```bash
chmod +x sleep-server.sh wake-server.sh
```

Fill in your own values (`SERVER_USER`, `SERVER_IP`, `SSH_KEY`, `SERVER_MAC`) before using them. `sleep-server.sh` is the same command wired into the systemd service in step 4 - the systemd unit is what makes it fire automatically on shutdown; the script itself is handy for testing or manual use.

### 8. Automatically wake the server whenever this machine boots

If you want the server to wake up every time you turn this PC on, create a matching service on the boot side:

```bash
sudo tee /etc/systemd/system/wake-server.service > /dev/null << 'EOF'
[Unit]
Description=Wake remote server on boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '/usr/bin/wakeonlan A0:D3:C1:2C:85:42 >> /var/log/wake-server.log 2>&1'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now wake-server.service
```

Replace the MAC address with your server's actual one (from step 6).

Unlike the shutdown service, this one is simpler - it just needs to run once at boot, after the network is up, so `ExecStart=` on its own is enough (no `/bin/true` trick, no `ExecStop=`).

Test:
```bash
cat /var/log/wake-server.log
```

**Note:** Wake-on-LAN reliably wakes a machine from suspend (S3). Waking a machine from a full poweroff/shutdown (S5) depends on BIOS/UEFI support - check for a setting like "Power On By PCI-E/PCIE" or "Wake on LAN" in the server's BIOS and make sure it's enabled, or WOL may only work intermittently.

## Testing

You can test the `ExecStop` trigger without a real reboot:

```bash
sudo systemctl stop notify-server-shutdown.service
cat /var/log/sleep-server.log
```

Then restart it (since `stop` leaves it inactive):
```bash
sudo systemctl start notify-server-shutdown.service
```

Finally, confirm with a real shutdown:
```bash
sudo systemctl poweroff
```

Then confirm the wake side with a real reboot:
```bash
sudo reboot
```
Check the server actually came back up, and check the log:
```bash
cat /var/log/wake-server.log
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Log file stays empty | `ExecStop=` missing a shell wrapper, or the unit never actually started/stopped |
| `journalctl -u notify-server-shutdown.service` shows `Missing '=', ignoring line` | The unit file got corrupted while editing (e.g. a long line got split by a text editor's auto-wrap) - rewrite it with a heredoc instead of a line editor |
| `Permission denied` writing to the log | Log file left over from an earlier test, owned by a different user than root |
| `Permission denied (publickey,password)` from SSH | The key used by the service (root) isn't the same key you tested manually as your own user, or the key has a passphrase |
| `network.target: Found ordering cycle` | `Before=` and `After=`/`Requires=` on the same targets contradict each other - drop the conflicting `Before=` lines |
| Service runs on shutdown but the command never fires | Check for a **duplicate/leftover unit file** with a similar name (e.g. an old `sleep-server.service` from earlier testing) - run `systemctl list-unit-files \| grep -iE "sleep\|notify\|wake"` to check for stragglers, and remove any that aren't the one you actually want |
| `wakeonlan` runs but server doesn't wake | WOL not enabled on the adapter (`ethtool -s <iface> wol g`), or the setting didn't persist across reboot |
| Server wakes once then stops waking | The BIOS/UEFI has its own WOL setting that needs enabling separately, or the `ethtool wol g` setting reset after a reboot/driver reload |
| WOL works on wired LAN but not remotely | Magic packets are broadcast on the local subnet - waking across routers/VLANs needs port forwarding (UDP 9) or a device on the same subnet to relay it |

Check service status and logs anytime with:
```bash
systemctl status notify-server-shutdown.service
systemctl status wake-server.service
journalctl -u notify-server-shutdown.service --no-pager
journalctl -u wake-server.service --no-pager
```
