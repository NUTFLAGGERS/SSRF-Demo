# Guest notes for the metadata demo

Debian 12 guest for a local AWS-style metadata simulation. The page states that this is a simulation and that it holds no cloud credentials. The guest has no `iam` path and no cloud keys.

Use these names in the commands below. Set them for the Proxmox host you use.

| Name | Meaning |
|:---|:---|
| `VMID` | Free Proxmox guest id |
| `STORAGE` | Storage pool for the disk and for cloud-init |
| `BRIDGE` | LAN bridge, often `vmbr0` |
| `GUEST_IP` | Free address on the LAN |
| `PREFIX` | LAN prefix length, such as `24` |
| `GATEWAY` | LAN gateway |
| `LAN_CIDR` | LAN network, such as `192.168.1.0/24` |

## VM

Leave the Proxmox datacenter firewall disabled. The guest can still have a firewall file. That file does not filter traffic while the datacenter firewall stays disabled. The guest packet filter is the active control.

SSH is open to `LAN_CIDR` only. Port `8080` is open to any source. Port `1338` listens on `127.0.0.1` only.

```bash
qm create VMID --name imds-demo --memory 1024 --balloon 0 --cores 1 --cpu host \
  --machine q35 --bios seabios --ostype l26 --scsihw virtio-scsi-single \
  --agent enabled=1 --vga serial0 --serial0 socket --rng0 source=/dev/urandom \
  --onboot 0 --net0 virtio,bridge=BRIDGE,firewall=1
qm importdisk VMID /var/lib/vz/template/iso/debian-12-generic-amd64.qcow2 STORAGE
qm set VMID --scsi0 STORAGE:vm-VMID-disk-0,discard=on,ssd=1
qm set VMID --ide2 STORAGE:cloudinit
qm set VMID --boot order=scsi0
qm set VMID --ciuser demo --sshkeys /root/imds-demo-ssh.pub \
  --ipconfig0 ip=GUEST_IP/PREFIX,gw=GATEWAY --nameserver GATEWAY
qm resize VMID scsi0 8G
qm start VMID
```

The login user is `demo`. The hostname is `imds-demo`. The address is `GUEST_IP`.

Log in with the same key that you copied to `/root/imds-demo-ssh.pub`. The account can run only the token switch:

```bash
sudo /usr/local/sbin/imds-token-only on
sudo /usr/local/sbin/imds-token-only off
```

`on` makes a plain metadata GET fail. Leave the switch `off` before a demo. The player path does not mint a token.

From a computer that is not on the LAN, run the switch on the Proxmox host:

```bash
qm guest exec VMID -- /usr/local/sbin/imds-token-only off
```

`qm guest exec` works after guest setup installs the guest agent. A non-interactive SSH session as `demo` can run the token switch, and it cannot read `/root/imds-demo-flag`.

## Services

Install the `imds-demo` source directory under `/opt/imds-demo`.

| Process | Bind | Role |
|:---|:---|:---|
| `imds-demo.service` | `0.0.0.0:8080` | Page with one URL box |
| `imds-metadata.service` | `127.0.0.1:1338` | Fake instance identity, placement, one event, and the ciphertext |

The page fetches a URL only when the host is `127.0.0.1`, the port is `1338`, and the scheme is `http`. Redirects stay off. Any other target is refused.

The ciphertext is base64 text. Decode it, then XOR each byte with the decrypt-key text from the event note. Repeat the key until the decoded bytes end. XOR uses the key text bytes. Do not decode the key as hex. The flag file on the guest is `/root/imds-demo-flag`. That file is not in the source directory. Replace it, then run:

```bash
python3 /opt/imds-demo/make_flag.py /root/imds-demo-flag /var/lib/imds-demo
chown root:imds /var/lib/imds-demo/decrypt-key /var/lib/imds-demo/ciphertext
chmod 0640 /var/lib/imds-demo/decrypt-key /var/lib/imds-demo/ciphertext
```

## Guest filter

`/etc/nftables.conf` drops new input except ICMP and TCP `22` from `LAN_CIDR`, plus TCP `8080` from any source. Port `8080` is open so a router forward can reach the page. A router forward keeps the visitor source address. A LAN-only rule on port `8080` drops that traffic. The metadata port stays on `127.0.0.1` only.

`/etc/pve/firewall/VMID.fw` can repeat an allow list for SSH, port `8080`, and ICMP. It does not filter traffic while the datacenter firewall stays disabled. Leave that firewall disabled. If you enable it with no host rules, it can block management of the host and the other guests.
