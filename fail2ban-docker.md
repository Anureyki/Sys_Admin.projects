# Pi-hole + Fail2ban on Docker

> Learning how to secure a Docker-based Pi-hole installation with Fail2ban, custom filters, and automated IP banning.

## Overview

I run Pi-hole in Docker as my network-wide DNS blocker, but one thing bothered me: the web interface had no protection against repeated login attempts.

If someone kept hammering the login page with bad passwords, Pi-hole would happily log the failures but wouldn't automatically block the source IP.

That's where Fail2ban comes in.

This project documents what I learned while integrating Fail2ban with a Docker-based Pi-hole container, including:

- Finding Docker log files
- Writing custom Fail2ban filters
- Creating a jail
- Testing bans
- Learning what `<<EOF` actually does
- Avoiding the temptation to guess Docker paths

The container used throughout this guide:

```text
pihole-unbound
```

---

## The Problem

My Pi-hole dashboard was accessible on the network:

```text
http://server-ip:8080/admin
```

However:

- No brute-force protection existed
- No rate limiting existed
- Failed logins were simply logged
- Attackers could repeatedly attempt passwords

Since I was already using Fail2ban for SSH, I wanted similar protection for Pi-hole.

The challenge was that Pi-hole was running inside Docker.

Fail2ban needs a log file to monitor, and Docker doesn't place logs in traditional locations such as:

```text
/var/log/
```

So the first task was finding where Docker was actually storing Pi-hole's logs.

---

# Step 1: Finding the Docker Log File

Instead of guessing, I asked Docker directly.

```bash
docker inspect --format='{{.LogPath}}' pihole-unbound
```

Output:

```text
/var/lib/docker/containers/<long-hash>/<long-hash>-json.log
```

This taught me an important lesson:

> Never guess Docker paths. Docker already knows where everything is.

Whenever I need a container's log file, I now use:

```bash
docker inspect --format='{{.LogPath}}' <container-name>
```

---

# Step 2: Creating the Fail2ban Filter

Fail2ban works by matching log entries against regular expressions.

After reviewing the Pi-hole logs, I found failed logins looked like this:

```text
 - Failed login attempt from 192.168.1.100
```

So I created a custom filter.

File:

```text
/etc/fail2ban/filter.d/pihole-auth.conf
```

Command:

```bash
sudo tee /etc/fail2ban/filter.d/pihole-auth.conf > /dev/null <<'EOF'
[Definition]
failregex = .* - Failed login attempt from <HOST>$
ignoreregex =
EOF
```

## What I Learned

This was my first time using a here document.

```bash
<<EOF
```

A here document lets you write multi-line text directly into a file from the command line.

Instead of:

```bash
nano filename
```

or

```bash
vim filename
```

I could generate the entire file from a single command.

That turned out to be surprisingly useful.

---

# Step 3: Creating the Fail2ban Jail

A filter only tells Fail2ban what to look for.

The jail tells Fail2ban:

- Which filter to use
- Which log file to monitor
- How many failures trigger a ban
- How long to ban an IP
- Which action to take

File:

```text
/etc/fail2ban/jail.d/pihole-auth.local
```

Command:

```bash
sudo tee /etc/fail2ban/jail.d/pihole-auth.local > /dev/null <<EOF
[pihole-auth]
enabled = true
port = http,https
filter = pihole-auth
logpath = /var/lib/docker/containers/long-hash/long-hash-json.log
maxretry = 5
bantime = 3600
findtime = 3600
action = iptables-multiport[name=pihole-auth, port="http,https", protocol=tcp]
EOF
```

## Jail Settings Explained

### maxretry

```text
maxretry = 5
```

Ban after 5 failed login attempts.

---

### bantime

```text
bantime = 3600
```

Ban for 1 hour.

---

### findtime

```text
findtime = 3600
```

Count failures occurring within a 1-hour period.

---

### action

```text
action = iptables-multiport
```

Use the Linux firewall to block the source IP.

---

# Step 4: Restarting Fail2ban

After creating the filter and jail:

```bash
sudo systemctl restart fail2ban
```

Then verify:

```bash
sudo fail2ban-client status
```

Expected output:

```text
Status
|- Number of jail: 2
`- Jail list: pihole-auth, sshd
```

Seeing `pihole-auth` appear next to `sshd` felt like a major win.

---

# Step 5: Testing the Jail

Configuration isn't finished until it's tested.

I intentionally failed authentication several times.

1. Opened:

```text
http://server-ip:8080/admin
```

2. Entered an incorrect password five times.

3. Checked Fail2ban:

```bash
sudo fail2ban-client status pihole-auth
```

Output:

```text
Banned IP list
```

My IP appeared in the list.

Success.

The jail was working exactly as intended.

---

# Key Takeaways

## Docker Logs Are Hidden in Plain Sight

Always use:

```bash
docker inspect --format='{{.LogPath}}'
```

Don't waste time guessing.

---

## Fail2ban Needs Two Things

### Filter

Defines what a failure looks like.

```text
/etc/fail2ban/filter.d/
```

### Jail

Defines what happens after a failure.

```text
/etc/fail2ban/jail.d/
```

---

## Here Documents Are Useful

I learned that:

```bash
<<EOF
```

lets me create entire configuration files from the terminal without opening an editor.

A small discovery, but one that immediately improved my workflow.

---

## Docker Log Paths Change

One important caveat:

If the container is recreated, Docker may generate a new log path.

That means:

```text
logpath =
```

inside the jail file may need updating.

Future improvement: automate this process.

---

## Always Test

Never assume a security configuration works.

Trigger the condition yourself.

Fail a login.

Check the jail.

Verify the ban.

Trust, but verify.

---

# Future Improvements

Ideas for future work:

- Automatically update Fail2ban jail log paths when containers are recreated
- Send alerts when IPs are banned
- Add trusted IP whitelists
- Store Fail2ban configuration in version control
- Expand protection to additional self-hosted services

---

# Screenshots

This repository includes screenshots showing:

- Docker log path discovery
- Initial Fail2ban errors
- Filter and jail creation
- Successful jail loading
- Working ban results

---

# Lessons Learned

This project started with a simple goal:

> Protect Pi-hole from brute-force login attempts.

Along the way I learned:

- How Docker stores logs
- How Fail2ban filters work
- How jails are configured
- How here documents work
- Why testing matters

The biggest lesson was simple:

> Don't guess. Inspect.

Docker knows where its logs are.

Fail2ban tells you exactly what's wrong.

Linux usually gives you the answer if you're willing to ask the right question.
