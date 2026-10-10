# Implement the metadata demo on Proxmox

Follow this guide in order on a Proxmox host. Proxmox is the program that runs the guests. The result is one Debian guest. The guest serves a local simulation of an AWS-style metadata service. The page states that it holds no cloud credentials.

Do not use Amazon EC2 Metadata Mock. That tool ships an `iam` category with key-shaped samples. This guest has no `iam` path, no key-shaped fields, and no cloud credentials.

Do not commit the flag or the decrypt key. Do not put either value in this guide.

## Names you must set

Set each name before the first command that uses it. Use the same values in every later step.

| Name | Meaning |
|:---|:---|
| `PROXMOX_HOST` | SSH name of the Proxmox host |
| `VMID` | Free guest id |
| `STORAGE` | Pool that can hold a disk and a cloud-init drive |
| `BRIDGE` | Bridge that carries the LAN |
| `GUEST_IP` | Free LAN address for the guest |
| `PREFIX` | Prefix length for that address |
| `GATEWAY` | LAN gateway |
| `LAN_CIDR` | LAN network in CIDR form |
| `PUBLIC_BASE` | Public page URL, such as `http://203.0.113.10:8080` |
| `SSH_PUB` | Public key file for the `demo` user |

A default Proxmox install uses the bridge `vmbr0`. Use another bridge when the LAN address is not on `vmbr0`.

## Rules for the finished guest

The page on port `8080` has one URL box. The guest fetches that URL only when the host is `127.0.0.1`, the port is `1338`, and the scheme is `http`. Redirects stay off. Any other target is refused.

The catalog listens on `127.0.0.1:1338` only. Do not publish port `1338`. A public forward of port `1338` does not serve the page.

The guest packet filter allows ICMP and TCP `22` from `LAN_CIDR`. It allows TCP `8080` from any source. A router forward keeps the visitor source address. A LAN-only rule on port `8080` drops those packets, and the public page then times out.

Leave the Proxmox datacenter firewall disabled. A guest firewall file does not filter traffic while that firewall is disabled. If you enable the datacenter firewall with no host rules, you can lose management of the host.

## 1. Test the source on the operator computer

Run this test in the `imds-demo` source directory before you create the guest. The test uses ports `13380` and `18080` on the operator computer. It does not use the flag you will place on the guest.

```bash
python3 check_local.py
```

Stop when the last line is not `local check passed`.

## 2. Record the host facts

Run these commands on the Proxmox host.

```bash
hostname
qm list
pvesm status
pve-firewall status
ip -4 addr show
```

1. Pick a `VMID` that is absent from `qm list`.
2. Pick a `STORAGE` pool that can store images.
3. Make sure that `pve-firewall status` reports the firewall as disabled. Do not enable it.
4. Pick a `GUEST_IP` that does not answer a ping. If it answers, choose another address.
5. Set `LAN_CIDR` to the network that may open SSH.

## 3. Prepare the image, the key, and the filter

On the Proxmox host, download the Debian 12 cloud image. The file is about 428 MB.

```bash
wget -q -O /var/lib/vz/template/iso/debian-12-generic-amd64.qcow2 \
  https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2
ls -lh /var/lib/vz/template/iso/debian-12-generic-amd64.qcow2
```

Copy `SSH_PUB` to `/root/imds-demo-ssh.pub` on the host.

In the source file `guest.nft`, set both `ip saddr` lines to `LAN_CIDR`. Leave the port `8080` rule as `tcp dport 8080 accept`, with no source limit.

## 4. Create the guest

Run this block on the Proxmox host. `qm importdisk` must finish before `qm set` attaches the disk. The imported volume name is `vm-VMID-disk-0`.

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
```

Write `/etc/pve/firewall/VMID.fw`. This file is inactive while the datacenter firewall stays disabled. The live filter is the guest file that `guest-setup.sh` installs.

```bash
cat > /etc/pve/firewall/VMID.fw << EOF
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
IN ACCEPT -source LAN_CIDR -p tcp -dport 22
IN ACCEPT -source LAN_CIDR -p tcp -dport 8080
IN ACCEPT -source LAN_CIDR -p icmp
EOF
qm start VMID
qm status VMID
```

`qm status VMID` must print `running`.

Wait until this command prints a shell prompt. Cloud-init can take about one minute. Do not use `qm guest exec` yet. The guest agent package is not installed yet.

```bash
ssh -i SSH_PRIV demo@GUEST_IP
```

`SSH_PRIV` is the private key that matches `SSH_PUB`.

## 5. Install the services

Copy the source files to the guest. Do not copy `__pycache__`. The required files are `banner.py`, `codec.py`, `make_flag.py`, `metadata_server.py`, `demo_server.py`, `imds-token-only`, `imds-metadata.service`, `imds-demo.service`, `demo-sudoers`, `guest.nft`, and `guest-setup.sh`.

```bash
scp -i SSH_PRIV -r imds-demo demo@GUEST_IP:/tmp/imds-demo
```

Write the flag to `/root/imds-demo-flag` before setup. The setup script exits when that file is missing or empty. The script does not contain the flag.

```bash
ssh -i SSH_PRIV demo@GUEST_IP 'sudo tee /root/imds-demo-flag >/dev/null'
```

Send the flag text on stdin, with one trailing newline. Then run setup. Debian cloud-init gives `demo` passwordless sudo. The setup script removes that broad sudo after it installs a sudo rule for the token switch only.

```bash
ssh -i SSH_PRIV demo@GUEST_IP \
  'sudo test -s /root/imds-demo-flag && echo flag-present && sudo DEBIAN_FRONTEND=noninteractive sh /tmp/imds-demo/guest-setup.sh'
```

Stop when the output does not contain `flag-present` and `guest setup complete`.

On the router, forward external TCP `8080` to `GUEST_IP` port `8080`. Do not use a public forward of port `1338` as part of the solve.

## 6. Validate the challenge path

Run this section from a computer that is not on `LAN_CIDR`, after the router forward is in place. Set `PUBLIC_BASE` to the public page URL. Turn token-only mode off first. This command runs on the Proxmox host, because the guest agent is installed now.

```bash
ssh root@PROXMOX_HOST "qm guest exec VMID -- /usr/local/sbin/imds-token-only off"
ssh root@PROXMOX_HOST "qm status VMID"
```

The status command must print `running`. The token command must finish with exit code `0`.

Define a helper. It posts one URL and prints the result box.

```bash
board() {
  curl -sS -X POST "$PUBLIC_BASE/" --data-urlencode "url=$1" \
  | python3 -c 'import sys,re,html; t=sys.stdin.read(); m=re.search(r"<pre>(.*?)</pre>",t,re.S); print(html.unescape(m.group(1)) if m else t)'
}
```

1. Fetch the page. Make sure that the body contains the simulation sentence and the comment `leftover health check`.

```bash
curl -sS "$PUBLIC_BASE/" | grep -F 'leftover health check'
```

2. Fetch a public URL through the box. Make sure that the result contains `Refused`.

```bash
board 'http://example.com/'
```

3. On the operator computer, fetch the catalog address directly. This command must fail. Port `1338` is not on that computer.

```bash
curl -sS -m 5 'http://127.0.0.1:1338/latest/meta-data/' || true
```

4. Fetch the index, then one value, through the box.

```bash
board 'http://127.0.0.1:1338/latest/meta-data/'
board 'http://127.0.0.1:1338/latest/meta-data/instance-id'
```

Make sure that the instance id result contains `i-demo903local`.

5. Walk the folders. A line that ends with `/` is a folder.

```bash
board 'http://127.0.0.1:1338/latest/meta-data/events/'
board 'http://127.0.0.1:1338/latest/meta-data/events/maintenance/'
board 'http://127.0.0.1:1338/latest/meta-data/events/maintenance/scheduled'
```

Make sure that the scheduled note contains a `decrypt-key` line and the address `http://127.0.0.1:1338/latest/dynamic/flag-blob`.

6. Fetch the blob through the box. Save the encoded line, not the simulation sentence, as `blob.txt`.

```bash
board 'http://127.0.0.1:1338/latest/dynamic/flag-blob'
```

7. Decrypt on the operator computer. Replace `KEY` with the decrypt-key text. XOR uses that text as bytes.

```bash
python3 -c 'import base64,sys; k=sys.argv[1].encode(); b=base64.b64decode(open("blob.txt").read()); print(bytes(x^k[i%len(k)] for i,x in enumerate(b)).decode())' 'KEY'
```

8. Compare that plaintext with the guest flag. The guest file has one trailing newline. The command must print `match`. It must not print the flag. Replace `PLAINTEXT` with the decrypt output.

```bash
ssh root@PROXMOX_HOST "qm guest exec VMID -- python3 -c \"import pathlib; t=pathlib.Path('/root/imds-demo-flag').read_text().strip(); print('match' if t=='PLAINTEXT' else 'differ')\""
```

9. Fetch an `iam` path through the box. Make sure that the result says `not found`. Make sure that the body has no `AKIA` text and no `Secret` text.

```bash
board 'http://127.0.0.1:1338/latest/meta-data/iam/'
```

10. From the operator computer, make sure that port `1338` on the guest LAN address does not accept a connection.

```bash
curl -sS -m 5 "http://GUEST_IP:1338/latest/meta-data/" || true
```

11. Turn token-only mode on. Fetch the instance id again. Make sure that the result says a plain GET does not get in. Then turn the mode off and make sure that the status is `off`.

```bash
ssh root@PROXMOX_HOST "qm guest exec VMID -- /usr/local/sbin/imds-token-only on"
board 'http://127.0.0.1:1338/latest/meta-data/instance-id'
ssh root@PROXMOX_HOST "qm guest exec VMID -- /usr/local/sbin/imds-token-only off"
ssh root@PROXMOX_HOST "qm guest exec VMID -- /usr/local/sbin/imds-token-only status"
```

Stop when any step misses the expected text. Leave token-only mode off when you stop.
