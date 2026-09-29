# Raspberry Pi 4 as a Tailscale exit node

Step-by-step setup for turning a Raspberry Pi 4 into a Tailscale exit node, with no port forwarding, no static IP, and no DDNS. I wrote up the reasoning and the CGNAT/NAT-traversal background [in a separate post](https://serkanyolun.com/writing). This repo is just the commands, kept as short as I could make them without leaving out the parts that actually cost me time.

## What you need

- A Raspberry Pi 4, wired with Ethernet (Wi-Fi works but you'll want the throughput headroom for exit node traffic)
- Ubuntu Server 26.04 LTS (that's what I ran this on, anything from 24.04 LTS up should be fine)
- A Tailscale account and a tailnet you're already part of

## 1. Install Tailscale and log in

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the link it prints to authenticate the device in your tailnet.

## 2. Apply the system settings

The Pi has to be allowed to forward traffic for other nodes, and while you're in there the congestion control and socket buffers are worth setting too. All of it lives in one sysctl file:

```bash
sudo cp config/99-tailscale.conf /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Do this before advertising the exit node. Recent Tailscale versions warn you if forwarding is off, but it's easy to scroll past, and the symptom on its own doesn't obviously point back at a sysctl setting: clients connect fine, traffic just doesn't go anywhere.

## 3. Advertise as an exit node

```bash
sudo tailscale set --advertise-exit-node
sudo tailscale up
```

This alone doesn't do anything yet. Go to the [admin console](https://login.tailscale.com/admin/machines), find the Pi, open its route settings, and approve it as an exit node. Advertising and approving are two separate steps, and until an admin approves it, advertising has no effect.

## 4. Use it from a client

On another machine in the same tailnet:

```bash
sudo tailscale set --exit-node=pi
```

`pi` here is the machine name from your tailnet. MagicDNS is what lets you use the name instead of the `100.x` address. To stop routing through it:

```bash
sudo tailscale set --exit-node=
```

On phones and in the desktop apps this is a toggle in the UI rather than a command. I used Android, iOS, Windows and macOS clients and didn't run into any behaviour differences between them.

To confirm it's actually working, check what the internet sees:

```bash
curl ifconfig.me
```

With the exit node selected, that should return your home connection's public IP rather than the address of whatever network you're currently sitting on. `tailscale status` tells you how the connection is carried: `direct` means peer-to-peer, `relay "xxx"` means it's going through a DERP relay.

## 5. UDP GRO for throughput

Out of the box, the Pi's routing throughput is lower than you'd expect from the hardware. Turning on UDP GRO forwarding makes a real difference:

```bash
sudo apt install ethtool   # skip if it's already installed
sudo ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

This setting doesn't survive a reboot on its own, so there's a systemd unit here to make it stick:

```bash
sudo cp config/tailscale-gro.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-gro.service
```

If your Pi's interface isn't `eth0`, check with `ip link` and edit the `ExecStart` line in the unit file before copying it over.

## 6. Turn off key expiry

By default, a node's key expires periodically and it has to re-authenticate. For a device you're not physically sitting in front of, that means losing remote access until you can plug in a keyboard. In the admin console, open the Pi's settings and disable key expiry (or set up an OAuth client / auth key if you'd rather automate re-auth than disable it).

## Known limitations

- **Direct isn't guaranteed.** Tailscale tries to build a peer-to-peer connection between the two nodes and falls back to a DERP relay when it can't get through the NATs on both sides. Relayed connections still work, they're just slower, and `tailscale status` is where you see which one you got.
- **Your upload speed is the real ceiling.** Every download that goes through the exit node rides your home connection's upload bandwidth back out. The Pi itself is rarely the bottleneck. Your ISP plan is.
- **Write your ACLs early.** The default tailnet policy is permissive. It's easy to get everything working first and tighten access later, but "later" has a way of not happening. Worth deciding who can use this exit node before you turn it on, not after.

## License

MIT, see [LICENSE](LICENSE).
