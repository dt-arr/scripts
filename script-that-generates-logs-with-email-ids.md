# Dynatrace Log Generator for OpenPipeline Masking Demonstration

This Markdown file contains a Bash script (`generate_logs_testnet2.sh`) that generates synthetic logs with email addresses to demonstrate how Dynatrace OpenPipeline can mask sensitive data, such as email addresses, for privacy and compliance. The script sends these logs to Dynatrace's log ingestion API, allowing users to test and showcase OpenPipeline's data masking capabilities. Below is a detailed description of the script's functionality, purpose, and instructions for usage.

## Script Purpose

The primary purpose of the script is to simulate the creation of application logs that include email addresses, mimicking real-world scenarios where sensitive data appears in logs. These logs are sent to Dynatrace to demonstrate how Dynatrace OpenPipeline can be configured to mask email addresses (e.g., replacing `william_shakespeare@example.com` with a redacted or anonymized version) to ensure compliance with data protection regulations (e.g., GDPR, CCPA). The script generates diverse log patterns with fictional email addresses, IP addresses, and Kubernetes attributes, providing a robust dataset for testing OpenPipeline's masking rules.

### Key Features
- **Log Generation**:
  - Generates logs every 10 seconds in one of eight predefined patterns, randomly selected.
  - Patterns include error logs, API request logs, business process logs, and HTTP access logs, each embedding an email address to simulate sensitive data.
  - Log attributes:
    - **Email IDs**: Randomly generated using names of famous people from the 16th–19th centuries (e.g., `jane_austen@example.com`), included in all log patterns to enable masking demonstrations.
    - **IP Addresses**:
      - Most patterns use IPs from TEST-NET-3 (`203.0.113.0/24`), reserved for documentation (RFC 5737).
      - The HTTP log pattern uses a server IP from TEST-NET-2 (`198.51.100.0/24`) and a client IP from TEST-NET-3.
    - **Correlation IDs**: 40-character strings with mixed uppercase/lowercase letters and numbers for unique log identification.
    - **Kubernetes Attributes**: Randomly assigns `k8s.container.name` as `sales-wrapper`, `onlineorders`, or `logs` to simulate containerized environments.
    - **Timestamps**: Randomly generated within the last hour, formatted as `YYYY-MM-DD HH:MM:SS` or HTTP log format (`DD/MMM/YYYY:HH:MM:SS +1000`).
- **Dynatrace Integration**:
  - Sends logs to Dynatrace’s log ingestion API (`https://<tenant>.live.dynatrace.com/api/v2/logs/ingest`) as JSON payloads.
  - Uses environment variables `DT_TENANT_ID` and `DT_API_TOKEN` for authentication.
  - Formats logs with `content`, `log.source` (`bash-script`), `k8s.container.name`, and ISO 8601 timestamp for compatibility with OpenPipeline.
- **Execution**:
  - Runs continuously, generating and sending logs every 10 seconds.
  - Supports background execution with output redirection to `script_output.log` to keep the terminal clean.
  - Allows retrieval to the foreground to view live console output (JSON payloads and status messages) for monitoring.
- **Safety and Compliance**:
  - Uses reserved IP ranges (TEST-NET-2: `198.51.100.0/24`, TEST-NET-3: `203.0.113.0/24`) and `example.com` domains (RFC 2606) to avoid conflicts with real networks.
  - Generates fictional email addresses to ensure no real personal data is used, making it safe for demonstrations.
- **OpenPipeline Demonstration**:
  - Check blog on how to mask it.

### How It Works
1. **Initialization**:
   - Validates `DT_TENANT_ID` and `DT_API_TOKEN` environment variables, exiting if unset to ensure proper Dynatrace authentication.
   - Defines a list of 10 famous people from the 16th–19th centuries (e.g., William Shakespeare, Charles Dickens) for email generation.
   - Specifies three Kubernetes container names (`sales-wrapper`, `onlineorders`, `logs`) for random assignment.
2. **Log Generation**:
   - Every 10 seconds, randomly selects one of eight log patterns, each including an email address.
   - Generates dynamic fields:
     - **Email**: Selects a random name from the famous people list and appends `@example.com` (e.g., `galileo_galilei@example.com`).
     - **IP Addresses**: Generates random IPs from TEST-NET-3 for most patterns; for the HTTP log (pattern 7), uses TEST-NET-2 for the server IP and TEST-NET-3 for the client IP.
     - **Correlation ID**: Creates a 40-character random string using `/dev/urandom` for log traceability.
     - **Timestamp**: Produces a recent timestamp within the last hour for realism.
     - **Container Name**: Randomly assigns one of the three container names as a log attribute.
   - Formats the log content according to the selected pattern, ensuring the email address is embedded for masking tests.
3. **POST to Dynatrace Generic Log Ingestion API**:
   - Constructs a JSON payload with the log `content`, `log.source`, `k8s.container.name`, and an ISO 8601 timestamp with nanosecond precision.
   - Sends the payload to Dynatrace using `curl`, authenticating with the API token in the `Authorization` header.
   - Outputs the generated log and Dynoccupied response status (success or failure) to the console or log file (`script_output.log`).
4. **Background Execution**:
   - Runs in the background using `&`, with output redirected to `script_output.log` to prevent terminal clutter.
   - Can be brought to the foreground with `fg` to view live logs or sent back to the background with `Ctrl+Z` and `bg`.
   - Supports detached execution with `nohup` for persistence across terminal sessions.

### Script Code
Below is the Bash script that generates and sends logs to Dynatrace for OpenPipeline masking demonstrations.

```bash
#!/bin/bash

# Check for required environment variables
if [ -z "$DT_TENANT_ID" ] || [ -z "$DT_API_TOKEN" ]; then
  echo "Error: DT_TENANT_ID and DT_API_TOKEN must be set as environment variables."
  exit 1
fi

# Dynatrace log ingestion endpoint
DT_LOG_INGEST_URL="https://${DT_TENANT_ID}.live.dynatrace.com/api/v2/logs/ingest"

# List of famous people from 16th-19th centuries
FAMOUS_PEOPLE=(
  "william_shakespeare"
  "leonardo_davinci"
  "galileo_galilei"
  "johannes_kepler"
  "isaac_newton"
  "benjamin_franklin"
  "samuel_johnson"
  "wolfgang_mozart"
  "jane_austen"
  "charles_dickens"
)

# Kubernetes container names
CONTAINER_NAMES=("sales-wrapper" "onlineorders" "logs")

# Function to generate a random 40-character correlation ID
generate_correlation_id() {
  cat /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 40
}

# Function to generate a random email from famous people
generate_email() {
  index=$((RANDOM % ${#FAMOUS_PEOPLE[@]}))
  echo "${FAMOUS_PEOPLE[$index]}@example.com"
}

# Function to generate a random IP from TEST-NET-3 (203.0.113.0/24)
generate_ip_testnet3() {
  last_octet=$((RANDOM % 256))
  echo "203.0.113.$last_octet"
}

# Function to generate a random IP from TEST-NET-2 (198.51.100.0/24)
generate_ip_testnet2() {
  last_octet=$((RANDOM % 256))
  echo "198.51.100.$last_octet"
}

# Function to generate a random timestamp (within the last hour for realism)
generate_timestamp() {
  # Generate timestamp within the last hour
  offset=$((RANDOM % 3600))
  date -u -d "@$(( $(date +%s) - $offset ))" "+%Y-%m-%d %H:%M:%S"
}

# Function to generate a random log pattern
generate_log() {
  # Randomly select a container name
  container_name=${CONTAINER_NAMES[$((RANDOM % ${#CONTAINER_NAMES[@]}))]}

  # Randomly select one of the 8 log patterns
  pattern=$((RANDOM % 8))
  email=$(generate_email)
  ip=$(generate_ip_testnet3)
  correlation_id=$(generate_correlation_id)
  timestamp=$(generate_timestamp)
  process_code="process-$(echo $email | tr '@' '-')-$(date +%s%N | head -c 13)"

  case $pattern in
    0)
      log_content="$timestamp [ERROR|[$ip] |$correlation_id |com.example.exmplstore.security.examplePersistentTokenRepository] Can't find credentials for series $email"
      ;;
    1)
      log_content="$timestamp [INFO |[$ip] |$correlation_id |class com.example.exmplfacades.order.exampleCheckoutLogger Checkout ABC] Receive API Request:method=placePayOrder, cartCode=302132310, customerID=$email|com.example.exmplstore.checkout.request.PayPlaceOrderRequestDto@3d3a53d5|]"
      ;;
    2)
      log_content="$timestamp [INFO |||com.example.exmpl.BusinessProcessLoggingAspect] Finish Action: [ PerformSubscriptionAction ], BusinessProcessCode: [ customerRegistrationProcess-$email-$(date +%s%N | head -c 13)], OrderCode:[ n/a ]"
      ;;
    3)
      log_content="$timestamp [INFO |||com.example.exmplmarketing.action.exampleSendCustomerNotificationAction] Successfully sent email forgottenPassword message for process forgottenPasswordProcess-$email-$(date +%s%N | head -c 13)"
      ;;
    4)
      log_content="$timestamp [INFO |[$ip] |$correlation_id|com.example.exmpl.UserDeleteInterceptor] Deleting userId: $email, actioned by userId: anonymous"
      ;;
    5)
      log_content="$timestamp [INFO |[$ip] |$correlation_id|com.example.exmpl.AdvantageApiClient] Loyalty: Check loyalty account exist for email: $email , wodCorrelationId 2fee137d-915e-46fb-b390-c69aae3f150"
      ;;
    6)
      log_content="$timestamp [INFO |||com.example.exmplbusproc.aop.BusinessProcessLoggingAspect] Begin Action: [ exampleAdvantageLinkEmailAction ], BusinessProcessCode: [ advantageLinkEmailProcess-$email-$(date +%s%N | head -c 13)], OrderCode:[ n/a ]"
      ;;
    7)
      server_ip=$(generate_ip_testnet2)
      log_content="$server_ip - - [$timestamp +1000] \"GET /reminder/$email/1 HTTP/1.1\" 200 90 \"https://www.example.com.au/my-account/order-details/306399901\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36\" $ip - $correlation_id 100"
      ;;
  esac

  # Create JSON payload for Dynatrace log ingestion
  json_payload=$(cat <<EOF
{
  "content": "$log_content",
  "log.source": "bash-script",
  "k8s.container.name": "$container_name",
  "timestamp": "$(date -u -d "$timestamp" --iso-8601=ns | sed 's/+00:00/Z/')"
}
EOF
)

  echo "$json_payload"
}

# Main loop: Generate and send logs every 10 seconds
while true; do
  log_json=$(generate_log)
  echo "Generated log: $log_json"

  # Send log to Dynatrace
  response=$(curl -s -X POST "$DT_LOG_INGEST_URL" \
    -H "Authorization: Api-Token $DT_API_TOKEN" \
    -H "Content-Type: application/json" \
    -d "$log_json")

  if [ $? -eq 0 ]; then
    echo "Log sent successfully to Dynatrace."
  else
    echo "Failed to send log to Dynatrace: $response"
  fi

  sleep 10
done
```

## Usage Instructions

### Prerequisites
1. **Environment Variables**:
   - `DT_TENANT_ID`: Your Dynatrace tenant ID (e.g., `abc12345` from `https://abc12345.live.dynatrace.com`).
   - `DT_API_TOKEN`: A Dynatrace API token with the `logs.ingest` scope.
   - Set in your terminal:
     ```bash
     export DT_TENANT_ID="your-tenant-id"
     export DT_API_TOKEN="your-api-token"
     ```
     To make persistent, add to your shell profile (e.g., `~/.bashrc`):
     ```bash
     echo 'export DT_TENANT_ID="your-tenant-id"' >> ~/.bashrc
     echo 'export DT_API_TOKEN="your-api-token"' >> ~/.bashrc
     source ~/.bashrc
     ```
2. **Dependencies**:
   - `curl`: For sending HTTP requests.
   - `date`: For timestamp generation.
   - `/dev/urandom`: For random string generation.
   - These are standard on most Linux/Unix systems.
3. **Dynatrace Setup**:
   - Ensure the API token has the `logs.ingest` permission.
   - Verify the tenant ID matches your Dynatrace environment.
   - Configure OpenPipeline in Dynatrace to mask email addresses (e.g., using regex patterns like `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`).

### Running the Script
1. Save the script as `generate_logs_testnet2.sh`:
   - Copy the script code above into a file named `generate_logs_testnet2.sh`.
2. Make it executable:
   ```bash
   chmod +x generate_logs_testnet2.sh
   ```
3. Run in the foreground (for testing):
   ```bash
   ./generate_logs_testnet2.sh
   ```
   - Outputs logs and status messages to the console every 10 seconds.
   - Press `Ctrl+C` to stop.
4. Run in the background with output redirection (recommended for demonstrations):
   ```bash
   ./generate_logs_testnet2.sh > script_output.log 2>&1 &
   ```
   - Redirects output to `script_output.log` to keep the terminal clean.
   - Displays job number and PID, e.g., `[1] 12345`.
5. Monitor the log file (optional):
   ```bash
   tail -f script_output.log
   ```
   - Shows real-time output; press `Ctrl+C` to stop monitoring.
6. Bring to foreground to view live logs:
   ```bash
   fg
   ```
   - Or `fg %1` if multiple jobs exist.
   - Displays live JSON payloads and Dynatrace response messages.
7. Send back to background:
   - Press `Ctrl+Z` to suspend, then:
     ```bash
     bg
     ```
8. Stop the script:
   ```bash
   fg
   ```
   - Press `Ctrl+C`, or:
     ```bash
     kill %1
     ```
   - Or use the PID:
     ```bash
     kill 12345
     ```

### Background Execution Notes
- **Output Redirection**: Redirecting to `script_output.log` captures all output (generated logs, JSON payloads, status messages, errors) for review, keeping the terminal usable.
- **Job Control**:
  - Use `jobs` to list background jobs.
  - Use `fg` to resume and view live console output for monitoring during a demo.
  - Use `bg` to resume in the background after suspension.
- **Detached Execution**: To run detached from the terminal (e.g., for long-running demos):
  ```bash
  nohup ./generate_logs_testnet2.sh > script_output.log 2>&1 &
  ```
  - Stop with `kill <PID>` after finding the PID via:
    ```bash
    ps aux | grep generate_logs_testnet2.sh
    ```


## Example Output
When running or brought to the foreground, the script outputs logs like:
```
Generated log: {"content":"2025-04-29 12:34:56 [INFO |[203.0.113.45] |K9p8Q3xJwZrStUv4XyZ7AbC2n5M6h8T1v0L2r4|com.example.exmpl.UserDeleteInterceptor] Deleting userId: william_shakespeare@example.com, actioned by userId: anonymous","log.source":"bash-script","k8s.container.name":"sales-wrapper","timestamp":"2025-04-29T12:34:56.000000000Z"}
Log sent successfully to Dynatrace.
```
For the HTTP log pattern:
```
Generated log: {"content":"198.51.100.123 - - [29/Apr/2025:12:34:56 +1000] \"GET /reminder/jane_austen@example.com/1 HTTP/1.1\" 200 90 \"https://www.example.com.au/my-account/order-details/306399901\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36\" 203.0.113.145 - K9p8Q3xJwZrStUv4XyZ7AbC2n5M6h8T1v0L2r4 100","log.source":"bash-script","k8s.container.name":"onlineorders","timestamp":"2025-04-29T12:34:56.000000000Z"}
Log sent successfully to Dynatrace.
```
In Dynatrace, with OpenPipeline masking, you might see:
```
Deleting userId: [REDACTED]@example.com, actioned by userId: anonymous
```

## Security Considerations
- **API Token**: Treat `DT_API_TOKEN` as sensitive. Avoid hardcoding or exposing it in scripts or public repositories.
- **Log File**: Secure `script_output.log`, as it contains unmasked email addresses (though fictional).
- **Fictional Data**: The script uses fictional email addresses and reserved IP ranges, ensuring no real personal data is generated.
- **Environment Variables**: Use a secrets manager or secure CI/CD pipeline for production-like environments.

## Troubleshooting
- **Environment Variables**: Ensure `DT_TENANT_ID` and `DT_API_TOKEN` are set; the script exits if unset.
- **Dynatrace Errors**: Check `script_output.log` for `curl` errors (e.g., invalid token, network issues).
- **Masking Issues**: Verify OpenPipeline rules match the email format and are applied to the correct log source or attribute.
- **Dependencies**: Confirm `curl`, `date`, and `/dev/urandom` are available on your system.

## Conclusion
This script provides a flexible and safe tool for generating logs with email addresses to demonstrate Dynatrace data masking capabilities. 