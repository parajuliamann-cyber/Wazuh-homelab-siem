# Wazuh-homelab-siem
Home SIEM lab build using Wazuh + docker on mac0S 
## OverView
Self hosted deployment of SIEM platform using Wazuhrunning on docker on Apple Silicon Mac.
Enrolled a host machine and checked if system correctly detects and logs suspicious activities

## WHY Docker
Due to Wazuh official pre-build VM only supports AMD64(x86_64) Architecture is incompatible with apple silicon (M-series) which uses ARM64 architecture thus due to such limitations docker was used using Wazuh official docker compose configuration

## Architecture
Wazuh stack runs as three containers 
-**Wazuh Indexer**- stores and indexes log/event data
-**Wazuh Manager**- receives data from agents and applies detection rules
-**Wazuh Dashboard**- web UI for viewing alerts and system status 

the host Mac runs Wazuh agent that sends system logs (authentication attempts , process and activity)to the manager over port-1514

## Setup steps & commands

**1.Install Docker desktop** for apple silicon build from docker.com

**2.Clone official Wazuh repo:**
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2
cd wazuh-docker/single-node
```

**3.generate SSL certificate** required for inter component communication
```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

**4.Launch the stack:**
```bash
docker compose up -d
```

**5. Verify all containers are running:**
```bash
docker ps
```

**6. Access the dashboard** at `https://localhost` (default credentials: `admin` / `SecretPassword`)

**7.Enroll the host Mac as an agent** via going through Endpoints Summary and deploying agent specifying macOS/arm64 and server address `127.0.0.1`


## Problems Encountered and fixes

After launching the stack Wazuh dashboard didn't load `https://localhost` 
Ran:
```bash
docker compose ps -a
```
and found out that the manager and dashboard were stuck in created and not starting

Checked log to investigate further:
```bash
docker compose logs wazuh.manager --tail 30
```
this gave idea that there were some missing SSL certificates file(`root-ca-manager.pem`).Looking into certs folder further confirmed that the file didn't exist

**Fix:**
Copied the current root CA certificate to generate missing file then loosened the file permissions
```bash
cp config/wazuh_indexer_ssl_certs/root-ca.pem config/wazuh_indexer_ssl_certs/root-ca-manager.pem
chmod -R 755 config/wazuh_indexer_ssl_certs/
docker compose up -d
```
ALL three containers started running after this

## Validating Detection
To confirm the system was actually detecting activity (not just running), intentionally triggered failed authentication attempts:
```bash
sudo whoami
# entered incorrect password 3-4 times on purpose
```
Checked the Wazuh dashboard under **Threat Hunting → Events** and confirmed the failed login attempts were correctly captured and logged as security events, proving the agent was sending real data and the manager was actively detecting suspicious activity.

<img width="1418" height="495" alt="Screenshot 2026-09-01 at 18 54 55" src="https://github.com/user-attachments/assets/8fa5ea07-42ad-4ad3-b911-309e56a40403" />


<img width="1458" height="825" alt="Screenshot 2026-09-01 at 18 51 25" src="https://github.com/user-attachments/assets/a98ceba4-b74e-4d29-b97d-a3dc8b0bd983" />

Additionally, enabled File Integrity Monitoring (FIM) by adding a watched directory to the agent's configuration file (`ossec.conf`):
```bash
sudo nano /Library/Ossec/etc/ossec.conf
```
Added the following inside the `<syscheck>` block:
```xml
<directories check_all="yes" realtime="yes">/Users/amanparajuli/Desktop</directories>
```
Restarted the agent to apply changes:
```bash
sudo /Library/Ossec/bin/wazuh-control restart
```
Verified FIM was working by creating and deleting test files on the Desktop and deletions were correctly captured as "File deleted" events in the dashboard

<img width="1447" height="704" alt="Screenshot 2026-09-01 at 18 57 38" src="https://github.com/user-attachments/assets/1c8858b1-627f-47c2-b3c9-2c4944c4d1c0" />

## Tools Used
Wazuh 4.9.2, Docker, Docker Compose, macOS (Apple Silicon - M4)







