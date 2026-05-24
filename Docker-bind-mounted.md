# How I Finally Stopped Re-Configuring Pi-hole Every Single Day

> & How I learned that the biggest threat to my self-hosted infrastructure was not hackers, DNS outages, or Docker. It was me.

## Overview

For a while, my Pi-hole installation felt cursed.

Not because Pi-hole was unstable.

Not because Docker was broken.

Not because Linux was failing me.

The problem was that every time I fixed one issue, I'd accidentally create another. Then I'd recreate a container, lose configuration, and spend the next hour rebuilding everything I had already configured three times before.

If you've ever found yourself re-adding the same adlists, regex filters, and blacklist entries over and over again, this README is for you.

This is the story of how I finally made my Pi-hole configuration persistent, portable, and recoverable.

---

# The Problem

Every day seemed to bring a new disaster.

## Port 53 Conflict

A Rocky Linux VM was installed using KVM.

Suddenly:

- Port 53 was occupied
- DNS stopped working
- Pi-hole could no longer bind to DNS

The culprit turned out to be `dnsmasq` running for the VM network.

---

## Port 80 Conflict

Then Pi-hole grabbed port 80.

Result:

- Nginx failed
- WordPress became inaccessible
- My web server effectively disappeared

I eventually moved Pi-hole's web interface to port 8080.

---

## The Worst Problem

Every time I recreated the container, I lost configuration.

That meant rebuilding:

### Adlists

- StevenBlack
- OISD
- HaGeZi
- SmartTV
- Apple
- TikTok
- Fake Shop blocklists

### Regex Filters

- Telemetry endpoints
- Analytics platforms
- Tracking pixels
- Ad servers

### Blacklisted Domains

```text
doubleclick.net
1forced.com
```

### Dashboard Settings

- Pi-hole password
- DNS preferences
- Web interface settings
- Custom configuration

Every.

Single.

Time.

At some point I realized I wasn't managing Pi-hole anymore.

I was repeatedly rebuilding Pi-hole.

---

# The Root Cause

Originally, I used Docker named volumes.

```yaml
volumes:
  pihole_data:
  pihole_dnsmasq:
```

On paper, this should have worked.

The volumes survived container deletion.

The problem was operational.

Sometimes I launched containers from different directories.

Sometimes I used different Compose files.

Sometimes I recreated containers quickly while troubleshooting.

The volumes still existed, but I wasn't always reconnecting them correctly.

Even worse:

- I couldn't easily browse them
- I couldn't easily back them up
- I couldn't easily verify what was stored

Docker knew where my data lived.

I didn't.

That's a problem.

---

# The Solution

I stopped relying on named volumes and switched to bind mounts.

Instead of letting Docker hide my configuration, I created dedicated host directories.

## Create Persistent Directories

```bash
sudo mkdir -p /opt/pihole/etc-pihole
sudo mkdir -p /opt/pihole/etc-dnsmasq.d
```

Now my Pi-hole configuration lives directly on the host.

I can see it.

I can back it up.

I can restore it.

I can inspect it whenever I want.

---

# Docker Compose Configuration

I rebuilt Pi-hole using bind mounts.

Example:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole-unbound
    restart: unless-stopped

    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80"

    environment:
      TZ: America/Chicago

    volumes:
      - /opt/pihole/etc-pihole:/etc/pihole
      - /opt/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
```

Notice there is no `version:` line and no named volumes.

Everything points directly to the host filesystem.

---

# Rebuilding the Container

After creating the new directories and Compose file, I deployed Pi-hole again:

```bash
docker compose up -d
```

Then I configured Pi-hole one final time:

- Added adlists
- Added regex filters
- Added blacklist entries
- Configured DNS settings
- Set dashboard password

The difference this time was that everything was being written directly to persistent host storage.

---

# Backing Everything Up

Immediately after finishing the configuration, I created a backup.

```bash
tar -czf pihole-backup.tar.gz \
/opt/pihole/etc-pihole \
/opt/pihole/etc-dnsmasq.d
```

This backup contains:

- Adlists
- Regex filters
- Blacklists
- Passwords
- DNS settings
- Pi-hole configuration

Now I can:

- Copy it to another drive
- Upload it to cloud storage
- Store it on a NAS
- Restore it after a server rebuild

For the first time, my configuration wasn't tied to a single container.

---

# The Result

Now my workflow is simple.

If the container dies:

```bash
docker rm -f pihole-unbound
```

My configuration remains untouched.

Everything still exists in:

```text
/opt/pihole/etc-pihole
```

and

```text
/opt/pihole/etc-dnsmasq.d
```

Recreating the container with:

```bash
docker compose up -d
```

brings everything back.

Instantly.

No re-entering adlists.

No rebuilding regex filters.

No retyping passwords.

No searching browser history for blacklist entries I forgot to document.

Just deploy and continue.

---

# Lessons Learned

## 1. Named Volumes Are Not Always the Best Choice

Named volumes are convenient.

But convenience can become a liability when configuration data is important.

If you frequently modify settings and need visibility into the files, bind mounts are often easier to manage.

---

## 2. Bind Mounts Give You Control

With bind mounts:

- You know where the data lives
- You can browse it
- You can back it up
- You can restore it

No Docker archaeology required.

---

## 3. Docker Compose Is Documentation

A good `docker-compose.yml` file is more than deployment automation.

It's infrastructure documentation.

Future you will be grateful.

---

## 4. Document Everything

Keep a record of:

- Adlists
- Regex filters
- Blacklists
- Custom DNS entries

Even if you think you'll never need it.

Because eventually you'll need it.

---

## 5. Infrastructure Should Survive Your Tinkering

Self-hosting isn't about building services that work.

It's about building services that continue working after you've inevitably "just made one quick change."

---

# Final Thoughts

This wasn't really a Pi-hole problem.

It was an infrastructure design problem.

I spent hours fixing symptoms:

- Port conflicts
- Broken containers
- Missing settings

The real solution was designing the system so it could survive container recreation, troubleshooting, and my own experimentation.

Taking one extra hour to implement proper persistence and backups saved me countless hours of rebuilding later.

Today my Pi-hole survives:

- Container deletion
- Docker updates
- Service restarts
- Host reboots

And if disaster strikes, I have a backup ready to go.

## The Final Lesson

People say:

> "If you do it right the first time, you'll never have to do it again."

In my case, it took about five rebuilds, several port conflicts, a disappearing configuration, and one stubborn home lab administrator.

But now I finally understand the system well enough that I probably won't have to do it again.

And that's the real goal of self-hosting:

**Not avoiding mistakes. Learning enough from them that you stop making the same one twice.**
