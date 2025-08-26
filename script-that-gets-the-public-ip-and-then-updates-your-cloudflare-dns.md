# Cloudflare Dynamic DNS Setup

Automatically update your Cloudflare DNS when your IP changes.

## What it does
- Checks your current public IP
- Compares it with your DNS record
- Updates Cloudflare DNS if IP changed
- Logs everything

## Setup

### 1. Get Cloudflare Info
- **Zone ID**: Go to your domain in Cloudflare dashboard → copy Zone ID from sidebar
- **API Key**: Go to My Profile → API Tokens → View Global API Key

### 2. Edit the Script
Change these variables:

```bash
auth_email="YOUR_EMAIL@EXAMPLE.COM"        # Your Cloudflare email
auth_key="YOUR_GLOBAL_API_KEY"             # Your API key
zone_identifier="YOUR_ZONE_ID"             # Your zone ID
record_name="subdomain.yourdomain.com"     # DNS record to update
```

### 3. Install
```bash
# Create script
sudo nano /opt/cloudflare-ddns.sh
# Paste the script content below
# Make executable
sudo chmod +x /opt/cloudflare-ddns.sh
# Test it
sudo /opt/cloudflare-ddns.sh
```

### 4. Auto-run with Cron
```bash
# Edit crontab
sudo crontab -e

# Add these lines
@reboot /bin/bash /opt/cloudflare-ddns.sh
*/5 * * * * /bin/bash /opt/cloudflare-ddns.sh >/dev/null 2>&1
```

## Check Logs
```bash
# View recent logs
sudo journalctl | grep "DDNS Updater" | tail -10
```

## Script

```bash
#!/bin/bash

# Cloudflare settings - CHANGE THESE
auth_email="YOUR_EMAIL@EXAMPLE.COM"
auth_key="YOUR_API_KEY"
zone_identifier="YOUR_ZONE_ID"
record_name="subdomain.yourdomain.com"
ttl=3600
proxy="false"

# Get current IP
ipv4_regex='([01]?[0-9]?[0-9]|2[0-4][0-9]|25[0-5])\.([01]?[0-9]?[0-9]|2[0-4][0-9]|25[0-5])\.([01]?[0-9]?[0-9]|2[0-4][0-9]|25[0-5])\.([01]?[0-9]?[0-9]|2[0-4][0-9]|25[0-5])'
ip=$(curl -s https://api.ipify.org)

if [[ ! $ip =~ ^$ipv4_regex$ ]]; then
    logger "DDNS Updater: Failed to get IP"
    exit 1
fi

# Get current DNS record
record=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$zone_identifier/dns_records?type=A&name=$record_name" \
    -H "X-Auth-Email: $auth_email" \
    -H "X-Auth-Key: $auth_key" \
    -H "Content-Type: application/json")

# Check if record exists
if [[ $record == *"\"count\":0"* ]]; then
    logger "DDNS Updater: Record $record_name not found"
    exit 1
fi

# Get old IP
old_ip=$(echo "$record" | sed -E 's/.*"content":"(([0-9]{1,3}\.){3}[0-9]{1,3})".*/\1/')

# Exit if IP hasn't changed
if [[ $ip == $old_ip ]]; then
    logger "DDNS Updater: IP $ip unchanged for $record_name"
    exit 0
fi

# Get record ID
record_id=$(echo "$record" | sed -E 's/.*"id":"([A-Za-z0-9_]+)".*/\1/')

# Update DNS record
update=$(curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$zone_identifier/dns_records/$record_id" \
    -H "X-Auth-Email: $auth_email" \
    -H "X-Auth-Key: $auth_key" \
    -H "Content-Type: application/json" \
    --data "{\"type\":\"A\",\"name\":\"$record_name\",\"content\":\"$ip\",\"ttl\":$ttl,\"proxied\":$proxy}")

# Check if update worked
if [[ $update == *"\"success\":true"* ]]; then
    logger "DDNS Updater: Updated $record_name to $ip"
else
    logger "DDNS Updater: Update failed for $record_name"
    exit 1
fi
```