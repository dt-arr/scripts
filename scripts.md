## Sending custom metrics to Dynatrace from a Linux machine every 15 seconds

In this section, you will find scripts that you would find useful

## Script approach

* This script extracts Wireless LAN signal strength using the `iwconfig` command 
* It extracts the system temperature from the file: `cat /sys/class/thermal/thermal_zone*/temp`
* Use `curl` to POST the above collected metrics to Dynatrace using the `/api/v2/metrics/ingest` endpoint
* It logs the output in JSON format because the result of the `curl` is in JSON format

## Pre-reqs
* DT API Token with metrics.ingest scope
* DT URL (<tenantid>.live.dynatrace.com)
* Optional: Entity Id of the host if you want to associate it with a particular host
  
## To start the script in CRON on reboot.

Make the shell script executable:
`chmod +x <script path`
Run `crontab -e` and add the following line:

```bash
@reboot /script-path.sh &

```


### Change the configuration variables and log directory to meet your needs

```bash
#!/bin/bash

# Ensure the PATH includes the directory for iwconfig
export PATH=$PATH:/usr/sbin:/sbin

# Configuration variables
API_TOKEN="dt0c01.<token>" # Token requires 
DYNATRACE_URL="https://<tenantid>.live.dynatrace.com/api/v2/metrics/ingest" # Enter the *live.dynatrace.com/api/v2/metrics/ingest
ENTITY_HOST="HOST-<entity-id>" # HOST Entity ID if needed to link the metric to the host

LOG_DIR="/home/pi/scripts/wlanlogs"
LOG_FILE="$LOG_DIR/signal_level_script_logs.log"

MAX_ENTRIES=60  # Adjusted for both wlan0 and wlan1 entries

# Function to log messages with timestamps
log_message() {
  local message=$1
  local timestamp=$(date +%s)
  echo "{\"timestamp\":$timestamp,\"message\":\"$message\"}" >> $LOG_FILE
}

while true; do
  # Extract signal levels for wlan0 and wlan1
  signal_level_wlan0=$(iwconfig wlan0 | grep -i --color signal | awk -F 'Signal level=' '{print $2}' | awk '{print $1}')
  if [ $? -ne 0 ]; then log_message "Failed to extract signal level for wlan0: $(iwconfig wlan0 2>&1)"; fi

  signal_level_wlan1=$(iwconfig wlan1 | grep -i --color signal | awk -F 'Signal level=' '{print $2}' | awk -F '/' '{print $1}')
  if [ $? -ne 0 ]; then log_message "Failed to extract signal level for wlan1: $(iwconfig wlan1 2>&1)"; fi

  # Extract the temperature, divide by 1000
  temperature=$(cat /sys/class/thermal/thermal_zone*/temp | awk '{sum+=$1} END {print sum/NR/1000}')
  if [ $? -ne 0 ]; then log_message "Failed to extract temperature: $(cat /sys/class/thermal/thermal_zone*/temp 2>&1)"; fi


  # Send the signal levels to Dynatrace and capture the responses // USE the localhost EEC option instead of directly sending out to the Dynatrace URL 
  response_wlan0=$(curl -s -L -X POST "$DYNATRACE_URL" \
  -H "Authorization: Api-Token $API_TOKEN" \
  -H 'Content-Type: text/plain' \
  --data-raw "network.signal.level.wlan0,dt.entity.host=$ENTITY_HOST $signal_level_wlan0")
  if [ $? -ne 0 ]; then log_message "Failed to send signal level for wlan0: $(curl -s -L -X POST '$DYNATRACE_URL' \
  -H 'Authorization: Api-Token $API_TOKEN' \
  -H 'Content-Type: text/plain' \
  --data-raw \"network.signal.level.wlan0,dt.entity.host=$ENTITY_HOST $signal_level_wlan0\" 2>&1)"; fi

  response_wlan1=$(curl -s -L -X POST "$DYNATRACE_URL" \
  -H "Authorization: Api-Token $API_TOKEN" \
  -H 'Content-Type: text/plain' \
  --data-raw "network.signal.level.wlan1,dt.entity.host=$ENTITY_HOST $signal_level_wlan1")
  if [ $? -ne 0 ]; then log_message "Failed to send signal level for wlan1: $(curl -s -L -X POST '$DYNATRACE_URL' \
  -H 'Authorization: Api-Token $API_TOKEN' \
  -H 'Content-Type: text/plain' \
  --data-raw \"network.signal.level.wlan1,dt.entity.host=$ENTITY_HOST $signal_level_wlan1\" 2>&1)"; fi

  response_temp=$(curl -s -L -X POST "$DYNATRACE_URL" \
  -H "Authorization: Api-Token $API_TOKEN" \
  -H 'Content-Type: text/plain' \
  --data-raw "system.temperature,dt.entity.host=$ENTITY_HOST $temperature")
  if [ $? -ne 0 ]; then log_message "Failed to send temperature: $(curl -s -L -X POST '$DYNATRACE_URL' \
  -H 'Authorization: Api-Token $API_TOKEN' \
  -H 'Content-Type: text/plain' \
  --data-raw \"system.temperature,dt.entity.host=$ENTITY_HOST $temperature\" 2>&1)"; fi


  # Get the current timestamp
  timestamp=$(date +%s)

  # Create JSON objects with the responses and timestamps
  log_entry_wlan0="{\"timestamp\":$timestamp,\"wlan_id\":\"wlan0\",\"response\":\"$response_wlan0\"}"
  log_entry_wlan1="{\"timestamp\":$timestamp,\"wlan_id\":\"wlan1\",\"response\":\"$response_wlan1\"}"

  # Log the JSON objects
  echo "$log_entry_wlan0" >> $LOG_FILE
  echo "$log_entry_wlan1" >> $LOG_FILE

  # Check the number of entries in the log file and trim if necessary
  entry_count=$(wc -l < $LOG_FILE)
  if [ $entry_count -gt $MAX_ENTRIES ]; then
    # Backup the original log file
    cp $LOG_FILE $LOG_FILE.bak

    # Keep only the latest MAX_ENTRIES entries
    tail -n $MAX_ENTRIES $LOG_FILE.bak > $LOG_FILE

    # Clean up the backup
    rm $LOG_FILE.bak
  fi

  # Wait for 15 seconds before the next iteration
  sleep 15
done 2>&1 | tee -a $LOG_FILE

```
