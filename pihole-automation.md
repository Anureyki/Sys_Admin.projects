# How I Automate Pi-hole Blocklist Updates (And Never Touch It Again)

## Introduction

One of the biggest advantages of running Pi-hole is that it quietly works in the background. The problem is that it still requires occasional maintenance.

Blocklists become outdated. New advertising domains appear constantly. And every now and then, the Pi-hole web interface decides it's taking an unscheduled vacation.

I found myself periodically logging into my server to update gravity, check container health, and restart services when something stopped responding.

That defeated the entire purpose of self-hosting infrastructure.

My goal became simple:

**Build a Pi-hole deployment that updates itself, recovers itself, and requires as little human intervention as possible.**

Like any good sysadmin, I wanted the server to solve its own problems while I slept.

---

## The Setup

My Pi-hole deployment runs inside Docker on an Ubuntu server using a container named:

```text
pihole-unbound
```

The container already had Docker's restart policy enabled:

```bash
--restart unless-stopped
```

That provided protection against:

- Host reboots
- Docker daemon restarts
- Unexpected container exits

But it didn't solve everything.

If Pi-hole's web interface stopped responding while the container itself remained running, Docker considered everything healthy.

I needed another layer of automation.

---

# 1. Daily Gravity Updates via Cron

Pi-hole uses Gravity to build its DNS blocking database from configured blocklists.

Without updates, the database gradually becomes stale.

Rather than remembering to manually run updates, I scheduled them automatically.

I opened my crontab:

```bash
crontab -e
```

Then added:

```cron
0 2 * * * /usr/bin/docker exec pihole-unbound pihole -g
```

### Why 2 AM?

I chose 2:00 AM because:

- Network activity is minimal
- No active users are affected
- Plenty of time exists for Gravity to complete

Every night, Pi-hole now refreshes its blocklists automatically.

No reminders.

No maintenance windows.

No forgotten updates.

Just fresh DNS filtering every day.

---

# 2. Auto-Restart Health Check Every 5 Minutes

The next problem was reliability.

Occasionally, the Pi-hole web interface would stop responding even though the container itself remained online.

Docker saw a running container.

Users saw a broken dashboard.

That's the kind of problem that can sit unnoticed for days.

So I created a lightweight health check using cron and curl.

I added another cron job:

```cron
*/5 * * * * /usr/bin/docker exec pihole-unbound curl -s -f -L -o /dev/null http://localhost/admin || /usr/bin/docker restart pihole-unbound
```

### How It Works

Every five minutes:

1. Cron executes curl inside the container.
2. curl attempts to reach the Pi-hole admin interface.
3. If the request succeeds, nothing happens.
4. If the request fails, Docker automatically restarts the container.

In plain English:

> "If the dashboard is broken, restart Pi-hole."

Simple.

Effective.

No complicated monitoring stack required.

---

# 3. Docker Restart Policy

I already had the container running with:

```bash
--restart unless-stopped
```

This gives another layer of resilience.

If:

- The host reboots
- Docker restarts
- The container crashes

Pi-hole automatically starts again.

Many people rely solely on Docker restart policies.

The problem is that Docker only knows whether a container is running.

It doesn't know whether the application inside the container is actually healthy.

That's why I combine Docker's restart policy with external health checks.

Together they cover different failure scenarios.

---

# Commands Reference

These are the exact commands I used to configure automation.

### Add Daily Gravity Update

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/bin/docker exec pihole-unbound pihole -g") | crontab -
```

### Add Health Check Auto-Restart

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/bin/docker exec pihole-unbound curl -s -f -L -o /dev/null http://localhost/admin || /usr/bin/docker restart pihole-unbound") | crontab -
```

### Verify Cron Jobs

```bash
crontab -l
```

### Verify Container Status

```bash
docker ps --filter name=pihole-unbound
```

---

# The Result

After implementing these automations, Pi-hole became almost entirely self-managing.

### Automatic Blocklist Updates

Gravity refreshes nightly without any manual intervention.

### Automatic Recovery

If the web interface crashes, the container recovers itself within five minutes.

### Host Reboot Protection

Docker ensures the service returns automatically after system restarts.

### Reduced Administrative Overhead

I haven't manually restarted Pi-hole in weeks.

That's exactly how infrastructure should behave.

If I'm constantly babysitting a service, it's not automated enough.

---

# Lessons Learned

This project reinforced several important sysadmin principles.

### Cron Is Still One of the Best Automation Tools

People love complicated orchestration systems.

Cron quietly continues doing useful work after decades.

Sometimes the simplest solution is the best solution.

### Health Checks Should Test the Service

A running container does not guarantee a working application.

Test the actual endpoint users depend on.

### Layered Resilience Works Better

Docker restart policies solve one class of failure.

External health checks solve another.

Combining them creates a more reliable system.

### You Don't Need Enterprise Monitoring for Everything

It's easy to over-engineer.

In this case, a single curl command provides meaningful monitoring and automatic recovery.

Not every problem requires Prometheus, Grafana, and seventeen dashboards glowing like a spaceship control panel.

---

# What's Next

There are a few improvements I plan to add in the future.

### Logging Restart Events

Track when and why automatic recoveries occur.

### Notifications

Send alerts through:

- Email
- Discord
- Telegram

whenever a restart is triggered.

### Automated Backups

Back up:

- Pi-hole configuration
- Gravity database
- Custom blocklists

before nightly updates run.

That would provide an easy rollback path if something ever goes sideways.

---

# Closing Thoughts

Automation isn't just about saving time.

It's about building systems that continue working when you're not watching them.

A self-hosted service should quietly perform its job without demanding constant attention.

By combining:

- Cron jobs
- Docker restart policies
- Lightweight health checks

I've turned my Pi-hole deployment into something far more resilient than the default setup.

The best automation is the automation you forget exists.

Because when your systems can recover themselves, update themselves, and stay healthy without intervention, you get to spend your time solving new problems instead of repeatedly fixing old ones.

Or, in the rarest of sysadmin outcomes, you get to sleep through the night without being summoned by a broken dashboard.
