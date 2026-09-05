# AWS SSM + Bash + Linux Fleet Automation Capstone

Provisioning and configuring a fleet of 5 Linux EC2 web servers, managed entirely through AWS Systems Manager - no SSH.

## Architecture

- **Compute:** 5x EC2 (Amazon Linux 2023), t3.micro, public subnet
- **Networking:** Default VPC, public subnet with auto-assigned public IPs, Internet Gateway + route table for public access
- **Security Group:** Inbound 80 (HTTP) and 443 (HTTPS) from `0.0.0.0/0`; no inbound port 22. Outbound left at default (all traffic allowed) for package installs and SSM connectivity
- **IAM:** `EC2-SSM-WebServer-Role` — trust policy allows `ec2.amazonaws.com` to assume the role; `AmazonSSMManagedInstanceCore` policy attached; wrapped in an instance profile and attached at launch
- **Tagging:** `Environment=Capstone`, `Project=LinuxAutomation`, `Role=WebServer`, `ManagedBy=SSM` — used for AND-combined tag targeting in Patch Manager and Run Command, so automation only ever touches this fleet
- **Management:** No SSH at any point. All configuration and patching is done via SSM Run Command and Patch Manager

## Bash Automation Script

`setup_webserver.sh` configures each instance, and is designed to run on either Amazon Linux or Ubuntu — detected at runtime via `/etc/os-release`.

**Sequence (dependency-ordered):**
1. Update/upgrade OS
2. Install web server (`httpd` on Amazon Linux, `nginx` on Ubuntu)
3. Create `cloudadmin` user
4. Add `cloudadmin` to the OS-appropriate admin group (`wheel` / `sudo`)
5. Create app directory (`/var/www/app`)
6. Set ownership and lock down permissions
7. Enable and start the web service
8. Open firewall ports (where a host firewall is present; otherwise the security group handles access control)
9. Validate that the service is actually listening on port 80

**Idempotency:** most commands used here (`yum install`, `apt-get install`, `usermod -aG`, `systemctl enable/start`, `mkdir -p`) are idempotent by default - safe to re-run without side effects. `useradd` is the one exception, and is wrapped in an `id cloudadmin` existence check before running.

```bash
#!/bin/bash
source /etc/os-release
set -e

if [[ "$ID" == "amzn" ]]; then
    PKG_INSTALL="yum install -y"
    WEB_PACKAGE="httpd"
    ADMIN_GROUP="wheel"
    echo "Detected Amazon Linux"
    yum update -y
    $PKG_INSTALL $WEB_PACKAGE
elif [[ "$ID" == "ubuntu" ]]; then
    PKG_INSTALL="apt-get install -y"
    WEB_PACKAGE="nginx"
    ADMIN_GROUP="sudo"
    echo "Detected Ubuntu"
    apt-get update -y && apt-get upgrade -y
    $PKG_INSTALL $WEB_PACKAGE
else
    echo "Unsupported OS: $ID"
    exit 1
fi

if id cloudadmin &>/dev/null; then
    echo "cloudadmin user already exists"
else
    useradd -m cloudadmin
    echo "Cloudadmin user is created"
fi

usermod -aG $ADMIN_GROUP cloudadmin
echo "cloudadmin user added to the admin group"

mkdir -p /var/www/app
echo "directory created"
chown cloudadmin:$ADMIN_GROUP /var/www/app
chmod 700 /var/www/app

systemctl enable $WEB_PACKAGE
systemctl start $WEB_PACKAGE

if command -v ufw &>/dev/null; then
    ufw allow 80/tcp
    ufw allow 443/tcp
    echo "ufw rules added"
else
    echo "ufw not installed; relying on security group for port control"
fi

if ss -tulnp | grep -q ':80'; then
    echo "Port 80 is listening - validation passed"
else
    echo "ERROR: Port 80 is not listening"
    exit 1
fi
```

## Deployment

1. Launch instances tagged with the 4 required tags, and attach `EC2-SSM-WebServer-Role`
2. Confirm all instances appear as **Online** in SSM Fleet Manager / Managed Nodes
3. Run **Patch Manager → Patch now (Scan)** targeting `tag:Project=LinuxAutomation` AND `tag:Role=WebServer` - confirm scan succeeds before installing
4. Run **Patch Manager → Patch now (Scan and install)** with the same targets
5. Run the script fleet-wide via **Systems Manager → Run Command → AWS-RunShellScript**, targeting the same tags
6. Verify: hit any instance's public IP over `http://` in a browser to confirm Apache/Nginx is serving

## Challenges Encountered

**`usermod` placement bug.** The admin-group assignment was originally placed inside the branch that only runs when `cloudadmin` already exists. On a brand-new instance, the user gets created via the `else` branch and the group assignment never ran. Confirmed by deleting the user and re-running the script to simulate a fresh server. **Fix:** moved `usermod -aG` outside the `if/else` entirely, since it's idempotent and safe to run unconditionally.

**Hardcoded group bug.** While the `usermod` line correctly used the `$ADMIN_GROUP` variable, the `chown` line further down had `sudo` hardcoded instead. This worked on Ubuntu (tested first) but failed on Amazon Linux with `chown: invalid group: 'cloudadmin:sudo'` on the first real fleet-wide run via SSM Run Command. **Fix:** replaced the hardcoded value with `$ADMIN_GROUP`.

**Stale SSM association.** After fixing a typo in a tag value (`Inux` → `Linux`), a Patch Manager "Patch now" execution kept reporting "No managed instances found" even though the corrected tags matched exactly. The existing association had cached the original (broken) target criteria and didn't refresh on its own. **Fix:** deleted the stale association and created a fresh one, which matched all 5 instances correctly.

## Known Limitations

- HTTPS (443) is open at the security group level but not actually configured on the web server — no TLS certificate or domain is set up, which was out of scope for this exercise
- No load balancer; each instance serves traffic directly on its own public IP
