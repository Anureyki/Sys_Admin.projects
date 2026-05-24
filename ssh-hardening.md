# How I Secured My Home Server Against SSH Brute-Force Attacks

## The Problem

One of the first things I did after setting up my Ubuntu home server was review the logs. What I found was both expected and unsettling.

My SSH service was being hammered by login attempts from IP addresses all over the world. Hundreds of failed login attempts appeared in the logs, originating from countries I had never interacted with and systems I certainly didn't authorize.

At the time, my server was still using password authentication for SSH access. While the password was reasonably strong, relying solely on passwords meant the server was vulnerable to automated brute-force attacks.

It wasn't a matter of **if** someone would try to break in.

It was already happening.

The question was whether my defenses would hold.

---

## The Discovery

To investigate further, I searched the SSH logs for failed authentication attempts:

```bash
sudo journalctl -u ssh | grep "Failed password"
```

The output was eye-opening.

Attempts were coming from IP addresses associated with locations across:

- China
- Russia
- United States
- Europe
- South America

The pattern quickly became obvious.

These weren't targeted attacks. They were automated bots scanning the public internet 24/7 looking for servers with weak credentials and default configurations.

The moment a server exposes SSH to the internet, it becomes part of a global background radiation of automated attack traffic.

Most administrators eventually discover this. The logs simply make it visible.

---

## The Solution

I decided to eliminate password-based authentication entirely and harden my SSH configuration.

### 1. Generate an SSH Key Pair

The first step was replacing passwords with cryptographic keys.

I generated a modern Ed25519 key pair:

```bash
ssh-keygen -t ed25519 -C "myemail"
```

After generating the key pair, I copied the public key to my server's authorized keys file:

```bash
~/.ssh/authorized_keys
```

Once configured, only devices possessing my private key could authenticate successfully.

No password guessing.

No brute-force attacks.

No credential stuffing.

Just cryptographic verification.

---

### 2. Harden SSH Configuration

Next, I modified:

```bash
/etc/ssh/sshd_config
```

and updated several security settings.

#### Disable Password Authentication

```text
PasswordAuthentication no
```

This completely disables password-based logins.

---

#### Disable Root Login

```text
PermitRootLogin no
```

Root accounts are one of the most commonly targeted usernames during SSH attacks.

Blocking direct root access removes an entire attack vector.

---

#### Enable Public Key Authentication

```text
PubkeyAuthentication yes
```

This ensures SSH accepts key-based authentication.

---

#### Restrict Login to a Specific User

```text
AllowUsers anureyki
```

Only my account is permitted to authenticate through SSH.

Any attempt to log in using other usernames is immediately rejected.

---

#### Disable Reverse DNS Lookups

```text
UseDNS no
```

This improves connection speed and reduces delays caused by DNS lookups during login attempts.

---

After making changes, I restarted SSH:

```bash
sudo systemctl restart ssh
```

---

### 3. Install and Configure fail2ban

Even though password authentication was disabled, I still wanted automated protection against persistent scanners and malicious clients.

I installed fail2ban:

```bash
sudo apt install fail2ban -y
```

fail2ban monitors log files and automatically blocks IP addresses that repeatedly fail authentication.

Checking status is simple:

```bash
sudo fail2ban-client status sshd
```

Benefits include:

- Automatic IP banning
- Reduced log spam
- Protection against future services that may expose authentication endpoints
- Lower resource consumption from repeated attack attempts

Watching bots get themselves banned is one of the few forms of entertainment a server administrator gets that doesn't involve staring at dashboards.

---

### 4. Optional: Change the SSH Port

Security through obscurity is not security.

However, reducing noise can still be useful.

I changed SSH from the default port:

```text
Port 22
```

to:

```text
Port 2222
```

in `sshd_config`.

Afterward, I updated UFW:

```bash
sudo ufw allow 2222/tcp
sudo ufw delete allow 22/tcp
```

and restarted SSH.

Changing ports won't stop a determined attacker, but it significantly reduces the number of automated bots blindly targeting port 22.

---

## Additional Network Protection with Pi-hole

Beyond securing SSH, I also run Pi-hole on my home network.

Pi-hole provides network-wide DNS filtering that blocks:

- Advertisements
- Tracking domains
- Telemetry endpoints
- Known malicious domains

While Pi-hole doesn't directly secure SSH, it serves as another layer in a defense-in-depth strategy.

Security works best when multiple systems work together.

SSH hardening protects server access.

fail2ban blocks abusive clients.

Pi-hole filters unwanted network traffic.

Each layer reduces risk.

---

## The Result

After implementing these changes:

### Password Login Disabled

Brute-force password attacks became effectively useless.

### Key-Only Authentication

Only devices with my private SSH key can authenticate.

### fail2ban Protection

Repeat offenders are automatically blocked.

### Reduced Noise

Changing the SSH port dramatically reduced automated scanning traffic.

### Stronger Overall Security

The server is now significantly more resilient against common internet-based attacks.

---

## Lesson Learned

One of the biggest misconceptions in home server administration is believing that nobody will notice your server.

The internet notices everything.

The moment a service becomes publicly accessible, automated scanners begin probing it.

Default SSH settings may work, but "works" and "secure" are not the same thing.

My biggest takeaways were:

- Don't leave SSH configured with defaults.
- Key-based authentication is non-negotiable.
- Disable anything you don't need.
- Monitor your logs regularly.
- Layer your defenses instead of relying on a single solution.

Most importantly:

**Security isn't a product you buy. It's a configuration you build.**

Every setting matters. Every service matters. Every exposed port matters.

A secure server isn't created by luck. It's created one deliberate configuration change at a time.
