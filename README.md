# SysAdmin Projects

A collection of real-world Linux system administration fixes, automation, and security hardening. Each project includes the problem, diagnostic steps, exact commands, and verification.

## Projects

### 1. [DNS Port Conflict](https://github.com/Anureyki/Sys_Admin.projects/blob/main/01%20DNS%20Filtering%20port%20conflicts)
Pi-hole vs Nginx fighting over port 80 — diagnosis, resolution, and remapping Pi-hole to port 8080.

### 2. [Fail2ban for Pi-hole](02-fail2ban-pihole/)
Brute force protection for Pi-hole's web interface using custom fail2ban filters and Docker log paths.

### 3. [SSH Hardening](03-ssh-hardening/)
Blocking brute force attacks with fail2ban, disabling root login, password authentication, and moving to key-only access.

### 4. [Pi-hole Automation](04-pihole-automation/)
Daily gravity updates and auto-restart health checks using cron and Docker restart policies.

### 5. [Docker Bind Mounts](05-docker-bind-mounts/)
Persistent storage for containerized services using Docker volumes and bind mounts.

## Tools Used
- Ubuntu Linux
- Docker / Podman
- Pi-hole (DNS filtering)
- Fail2ban (intrusion prevention)
- Git & GitHub (version control)
- Cron (scheduled tasks)
- UFW (firewall)

## About
Each project is documented with:
- The problem I encountered
- Diagnostic commands used
- Step-by-step solutions
- Verification commands
- Lessons learned

These are real troubleshooting scenarios from my home lab, not theoretical exercises.

## Connect
- GitHub: [github.com/Anureyki](https://github.com/Anureyki)
- Portfolio: [github.com/Anureyki/Sys_Admin.projects](https://github.com/Anureyki/Sys_Admin.projects)
