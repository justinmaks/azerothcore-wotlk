# AzerothCore WoW Server Setup Guide

## Server Specifications

- **vCPU:** 4 cores
- **RAM:** 6 GB
- **Disk:** 100 GB
- **OS:** Ubuntu/Linux
- **Capacity:** Comfortable for 10-50 players

---

## Prerequisites

- Docker and Docker Compose installed
- Ubuntu/Linux server with root access
- UFW firewall (or equivalent)
- WoW 3.3.5a (WotLK) client

---

## Step 1: Create Non-Root User

```bash
# Create acore user
adduser acore

# Add user to docker group
usermod -aG docker acore

# Apply group changes (logout/login or use newgrp)
newgrp docker
```

**Why:** Docker containers expect user ID 1000, running as root causes permission conflicts.

---

## Step 2: Configure Firewall

```bash
# Allow game server port
sudo ufw allow 8085/tcp

# Allow authentication server port
sudo ufw allow 3724/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

**Port Reference:**
- **3724/tcp** - Authentication server (login)
- **8085/tcp** - World server (game traffic)
- **3306/tcp** - MySQL (internal only, DO NOT open externally)

---

## Step 3: Clone and Setup AzerothCore

```bash
# Switch to acore user
su - acore

# Clone repository
cd ~
git clone https://github.com/azerothcore/azerothcore-wotlk.git
cd azerothcore-wotlk

# Copy docker compose configuration
cp conf/dist/docker-compose.yml docker-compose.yml
```

---

## Step 4: Build and Start Services

```bash
# Build Docker images (takes 10-20 minutes)
docker compose build

# Start all services
docker compose up -d
```

**Services Started:**
- `ac-database` - MySQL database server
- `ac-authserver` - Authentication server
- `ac-worldserver` - Game world server
- `ac-db-import` - Database initialization
- `ac-client-data-init` - Game data initialization

---

## Step 5: Verify Installation

```bash
# Check running containers
docker ps

# Check logs for any errors
docker logs ac-authserver
docker logs ac-worldserver
docker logs ac-database

# Check database import
docker logs ac-db-import
```

**All containers should show as "Up" or "Exited (0)" for initialization containers.**

---

## Step 6: Update Database Realmlist

**CRITICAL:** You must update the realmlist address in the database or clients won't see your server.

```bash
# Get your server's public IP
curl ifconfig.me

# Connect to the database
docker exec -it ac-database mysql -uroot -ppassword

# Update the realmlist (replace with YOUR IP)
USE acore_auth;
SELECT * FROM realmlist;
UPDATE realmlist SET address='YOUR_SERVER_PUBLIC_IP' WHERE id = 1;
SELECT * FROM realmlist;
exit;

# Restart worldserver to apply changes
docker compose restart ac-worldserver
```

**Example:**
```sql
-- If your server IP is 45.123.456.789
UPDATE realmlist SET address='45.123.456.789' WHERE id = 1;
```

---

## Step 7: Create Admin Account

```bash
# Connect to worldserver console
docker attach ac-worldserver

# Inside the console, create account
account create <username> <password>

# Set account to GM level 3 (full admin)
account set gmlevel <username> 3 -1

# Detach from console (CTRL+P then CTRL+Q)
```

---

## Step 8: Final Verification Checklist

Before connecting with your client, verify everything is set up correctly:

### Server Status
```bash
# All containers should be running
docker ps

# Expected output: ac-database, ac-authserver, ac-worldserver all "Up"
```

### Firewall Check
```bash
sudo ufw status

# Should show:
# 3724/tcp                   ALLOW       Anywhere
# 8085/tcp                   ALLOW       Anywhere
```

### Database Realmlist Check
```bash
docker exec -it ac-database mysql -uroot -ppassword -e "SELECT id, name, address, port FROM acore_auth.realmlist;"

# Verify:
# - address matches your server's public IP
# - port is 8085
```

### Service Logs Check
```bash
# Check for errors in authserver
docker logs ac-authserver --tail 50

# Check for errors in worldserver
docker logs ac-worldserver --tail 50

# Both should show no critical errors
```

### Network Connectivity Test
```bash
# Test auth server port (from another machine or use online port checker)
telnet YOUR_SERVER_IP 3724

# Test world server port
telnet YOUR_SERVER_IP 8085

# Or use netcat if telnet not available:
nc -zv YOUR_SERVER_IP 3724
nc -zv YOUR_SERVER_IP 8085
```

### Client Realmlist Setup
On your **client machine** (not server):

1. Navigate to WoW 3.3.5a folder
2. Edit `Data/enUS/realmlist.wtf`
3. Content should be:
   ```
   set realmlist YOUR_SERVER_PUBLIC_IP
   ```
4. Save and close

### Test Connection

1. Launch WoW 3.3.5a client
2. You should see your realm listed
3. Login with the account you created
4. Create a character and enter the world

**If realm shows "Offline":**
- Check worldserver is running: `docker ps`
- Verify database realmlist matches your IP
- Check firewall allows port 8085
- Review worldserver logs: `docker logs ac-worldserver`

---

## Quick Start Summary

Once everything is set up, this is all you need to do:

```bash
# Start server
cd ~/azerothcore-wotlk
docker compose up -d

# Stop server
docker compose down

# View logs
docker logs ac-worldserver -f

# Create accounts
docker attach ac-worldserver
# Then: account create username password
# CTRL+P, CTRL+Q to detach
```

---

## Realmlist Configuration

The realmlist tells your WoW client where to connect. You need to configure it in **two places**.

### 1. Client-Side Realmlist (Required)

**For Windows:**
1. Navigate to your WoW 3.3.5a installation folder
2. Open `Data/enUS/realmlist.wtf` (or `enGB`, `deDE`, etc. based on your client locale)
3. Replace the entire content with:
   ```
   set realmlist YOUR_SERVER_IP
   ```
4. Save the file
5. Launch WoW

**For Mac:**
1. Navigate to `/Applications/World of Warcraft/Data/enUS/`
2. Edit `realmlist.wtf`
3. Set to: `set realmlist YOUR_SERVER_IP`

**Examples:**
```
set realmlist 192.168.1.100        # Local network
set realmlist 45.123.456.789       # Public IP
set realmlist myserver.example.com # Domain name
```

**Note:** Do NOT include the port (8085) in the realmlist file - the client automatically uses the correct ports.

### 2. Database Realmlist (Required)

The server's database also needs to know its own IP address so it can tell clients where to connect.

```bash
# Connect to the database
docker exec -it ac-database mysql -uroot -ppassword

# Switch to auth database
USE acore_auth;

# Check current realmlist
SELECT * FROM realmlist;

# Update the realm address to your server IP
# IMPORTANT: Use your public IP if hosting on internet, or local IP if LAN only
UPDATE realmlist SET address = 'YOUR_SERVER_IP' WHERE id = 1;

# Optionally update the realm name
UPDATE realmlist SET name = 'My Custom Server' WHERE id = 1;

# Verify the change
SELECT * FROM realmlist;

# Exit
exit;
```

**After updating, restart the worldserver:**
```bash
docker compose restart ac-worldserver
```

**Getting your server IP:**
```bash
# Get public IP (for internet hosting)
curl ifconfig.me

# Get local network IP (for LAN)
hostname -I
```

### Realmlist Table Explained

The `realmlist` table in the `acore_auth` database contains:

| Column | Description | Example |
|--------|-------------|---------|
| `id` | Realm ID | 1 |
| `name` | Realm name shown in client | "AzerothCore" |
| `address` | **Server IP** (MUST match your server) | "45.123.456.789" |
| `localAddress` | LAN IP (127.0.0.1 for local) | "127.0.0.1" |
| `localSubnetMask` | Subnet mask | "255.255.255.0" |
| `port` | World server port | 8085 |
| `icon` | Realm type icon | 0 (PvP), 1 (Normal), 6 (RP) |
| `flag` | Realm status | 0 (offline), 2 (online) |
| `timezone` | Timezone | 1 |
| `allowedSecurityLevel` | Min account level | 0 (anyone) |
| `population` | Population display | 0 (auto) |

### Common Realmlist Scenarios

**Local Testing (same machine):**
```bash
# Client realmlist.wtf
set realmlist 127.0.0.1

# Database
UPDATE realmlist SET address = '127.0.0.1' WHERE id = 1;
```

**Local Network (LAN):**
```bash
# Client realmlist.wtf (use server's local IP)
set realmlist 192.168.1.100

# Database
UPDATE realmlist SET address = '192.168.1.100' WHERE id = 1;
```

**Public Server (Internet):**
```bash
# Client realmlist.wtf (use server's public IP)
set realmlist 45.123.456.789

# Database
UPDATE realmlist SET address = '45.123.456.789' WHERE id = 1;
```

**Using Domain Name:**
```bash
# Client realmlist.wtf
set realmlist wow.myserver.com

# Database
UPDATE realmlist SET address = 'wow.myserver.com' WHERE id = 1;
```

### Changing Realm Name and Type

```bash
# Connect to database
docker exec -it ac-database mysql -uroot -ppassword
USE acore_auth;

# Change realm name
UPDATE realmlist SET name = 'My Epic Server' WHERE id = 1;

# Change realm type
# 0 = PvP (red icon)
# 1 = Normal/PvE (green icon)
# 4 = PvP-RP
# 6 = RP (blue icon)
# 8 = PvE-RP
UPDATE realmlist SET icon = 1 WHERE id = 1;

# Restart worldserver for changes to take effect
exit;
```

```bash
docker compose restart ac-worldserver
```

### Troubleshooting Realmlist

**Problem: "No realms available" or server not showing:**

1. **Check database realmlist:**
   ```bash
   docker exec -it ac-database mysql -uroot -ppassword -e "SELECT * FROM acore_auth.realmlist;"
   ```

2. **Verify IP matches your server:**
   - If hosting publicly, use your public IP
   - If on LAN, use local network IP
   - Get your public IP: `curl ifconfig.me`
   - Get your local IP: `ip addr` or `hostname -I`

3. **Ensure worldserver is running:**
   ```bash
   docker ps | grep worldserver
   docker logs ac-worldserver
   ```

4. **Check firewall allows port 8085:**
   ```bash
   sudo ufw status | grep 8085
   ```

5. **Verify client realmlist.wtf is correct:**
   - Must match database address
   - No typos or extra spaces
   - File must be saved after editing

**Problem: Can see realm but can't connect:**

- Firewall blocking port 8085
- Realmlist IP doesn't match actual server IP
- Client version mismatch (must be 3.3.5a build 12340)

**Problem: Realm shows as "Offline":**

- Worldserver not running: `docker logs ac-worldserver`
- Database connection issue
- Wrong port configured (should be 8085)

### Testing Your Realmlist

```bash
# Test if auth server is reachable
telnet YOUR_SERVER_IP 3724

# Test if world server is reachable  
telnet YOUR_SERVER_IP 8085

# If telnet not installed:
nc -zv YOUR_SERVER_IP 3724
nc -zv YOUR_SERVER_IP 8085
```

If both ports respond, your realmlist configuration is likely correct.

---

## Useful Commands

### Service Management

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart specific service
docker compose restart ac-worldserver

# View logs
docker compose logs -f ac-worldserver
```

### Database Access

```bash
# Connect to MySQL
docker exec -it ac-database mysql -uroot -ppassword

# Databases
# - acore_auth: Authentication and accounts
# - acore_characters: Character data
# - acore_world: Game world data
```

### Server Console

```bash
# Attach to worldserver console
docker attach ac-worldserver

# Detach without stopping (CTRL+P, CTRL+Q)

# Common commands in console:
# - account create <user> <pass>
# - account set gmlevel <user> <level> -1
# - server shutdown <seconds>
# - reload config
```

---

## Monitoring and Maintenance

### Check Resource Usage

```bash
# System resources
htop

# Docker container stats
docker stats

# Free memory
free -h

# Disk usage
df -h
```

### Backup Database

```bash
# Backup all databases
docker exec ac-database mysqldump -uroot -ppassword --all-databases > backup.sql

# Restore from backup
docker exec -i ac-database mysql -uroot -ppassword < backup.sql
```

---

## Troubleshooting

### Permission Issues

If you see permission errors:

```bash
# Fix ownership
sudo chown -R 1000:1000 ~/azerothcore-wotlk/env/dist/etc
sudo chown -R 1000:1000 ~/azerothcore-wotlk/env/dist/logs
```

### Database Import Failed

```bash
# Check logs
docker logs ac-db-import

# Retry import
docker compose up ac-db-import

# Clean restart
docker compose down
docker volume rm azerothcore-wotlk_ac-database
docker compose up -d
```

### Can't Connect to Server

1. Verify firewall ports are open: `sudo ufw status`
2. Check services are running: `docker ps`
3. Verify realmlist IP is correct
4. Check server logs: `docker logs ac-worldserver`
5. Verify database realmlist: `SELECT * FROM acore_auth.realmlist;`

### High Memory Usage

```bash
# Check what's using memory
docker stats

# Restart services if needed
docker compose restart
```

---

## Performance Optimization

### Recommended Settings

For 10-50 players on your specs (4 vCPU, 6GB RAM):
- Default settings should work well
- Monitor with `docker stats` and `htop`
- Database is usually the bottleneck

### Configuration Files

Located in `~/azerothcore-wotlk/env/dist/etc/`:
- `worldserver.conf` - World server settings
- `authserver.conf` - Auth server settings

Edit and restart services to apply changes.

---

## Security Best Practices

1. **Never expose MySQL port 3306** to the internet
2. **Change default MySQL password** in docker-compose.yml
3. **Use strong passwords** for admin accounts
4. **Keep Docker and system updated**
5. **Regular backups** of database
6. **Monitor logs** for suspicious activity
7. **Limit SSH access** (consider key-based auth only)

---

## Updating AzerothCore

```bash
# Stop services
docker compose down

# Pull latest changes
git pull

# Rebuild images
docker compose build

# Start services
docker compose up -d
```

---

## Support and Resources

- **AzerothCore Wiki:** https://www.azerothcore.org/wiki/
- **Discord Community:** https://discord.gg/gkt4y2x
- **GitHub Repository:** https://github.com/azerothcore/azerothcore-wotlk
- **Database Documentation:** https://www.azerothcore.org/wiki/database-world

---

## Quick Reference

| Component | Port | Purpose |
|-----------|------|---------|
| Auth Server | 3724 | Player authentication |
| World Server | 8085 | Game world connection |
| MySQL | 3306 | Database (internal only) |

**Admin Commands:**
- Create account: `account create <user> <pass>`
- Set GM level: `account set gmlevel <user> 3 -1`
- Teleport: `.tele <location>`
- Add item: `.additem <itemID>`
- Level up: `.levelup <amount>`

---

## Notes

- Server capacity: ~50 players comfortably on current specs
- Expansion: WotLK (3.3.5a)
- Database auto-updates on container restart
- Logs stored in: `~/azerothcore-wotlk/env/dist/logs/`
