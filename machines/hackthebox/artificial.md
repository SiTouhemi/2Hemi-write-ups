# Artificial

### Machine Information

* **Target IP**: 10.129.255.138
* **Attacker IP**: 10.10.14.99
* **Difficulty**: Medium
* **OS**: Linux

### Executive Summary

This machine involved exploiting a Keras model deserialization vulnerability through Lambda layers to achieve Remote Code Execution (RCE). The attack chain progressed from initial web exploitation to privilege escalation via credential discovery and backup file analysis.

### Reconnaissance & Enumeration

Initial reconnaissance revealed a web application utilizing AI/ML models built with Keras/TensorFlow.

### Vulnerability Analysis

#### Keras Lambda Layer Deserialization

Through research, I discovered that Keras models using Lambda Layers are vulnerable to deserialization attacks that can lead to RCE.

**Reference**: [ProtectAI Knowledge Base - Deserialization Threats](https://protectai.com/insights/knowledge-base/deserialization-threats/PAIT-KERAS-101)

**Key vulnerability characteristics**:

* Deserialization threats can lead to unauthorized code execution in AI models
* Keras models using Lambda Layers are particularly vulnerable
* Attacks can result in data theft, system compromise, and broader security breaches

### Exploitation

#### Step 1: Exploit Development

Located a proof-of-concept exploit for TensorFlow RCE:

* **Exploit Source**: https://github.com/Splinter0/tensorflow-rce/blob/main/exploit.py

#### Step 2: Docker Environment Setup

The exploit required a specific Docker environment with compatible TensorFlow versions:

```bash
# Start Docker service
sudo systemctl start docker

# Build the exploit container
docker build -t my-exploit .

# Run container with volume mapping
docker run -it --rm -v "$PWD":/app -w /app my-exploit

# Install dependencies
pip install -r requirements.txt

# Generate malicious .h5 model
python3 exploit.py
```

#### Step 3: Payload Delivery

Uploaded the malicious `exploit.h5` file to the target application and triggered the deserialization vulnerability by accessing the model, resulting in a reverse shell.

#### Step 4: Post-Exploitation Enumeration

Upgraded the shell for better interaction:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash");'
```

Discovered multiple SQLite databases containing sensitive information:

* `/home/app/instance/users.db`
* `/opt/backrest/tasklogs/logs.sqlite`
* `/opt/backrest/oplog.sqlite`

#### Step 5: Credential Extraction

Extracted user credentials from `users.db`:

* Multiple user accounts with hashed passwords
* Focused on `@artificial.htb` domain users to avoid interference with other players

**Cracked passwords**:

* `aa` (test account)
* `mattp005numbertwo`
* `marwinnarak043414036`

#### Step 6: SSH Access

Used Hydra to test credentials against SSH for user `gael` ( i just wanted to use hydra):

```bash
hydra -l gael -P passwords.txt ssh://artificial.htb
```

**Result**: Successfully authenticated as `gael:mattp005numbertwo`

### Privilege Escalation

#### Backrest Service Analysis

Discovered a `backrest` binary running a web server. While direct access to the service directory was restricted, backup files were accessible.

#### Backup Analysis

Extracted and analyzed backrest backup files:

```bash
# Extract backup
unzip backup.zip

# Search for credentials
grep -r "password" .
```

Found base64-encoded bcrypt hash for `backrest_root`:

```json
{
  "name": "backrest_root", 
  "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
}
```

#### Password Cracking

```bash
# Decode base64 and save hash
echo "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP" | base64 -d > hash

# Crack with John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt hash
```

**Result**: `backrest_root:!@#$%^`

### Lessons Learned & Mistakes Analysis

#### Areas for Improvement in Documentation:

1. **Missing Network Mapping**: No port scan results or service enumeration details
2. **Incomplete Initial Access**: How was the web application discovered? What services were running?
3. **Screenshot Integration**: Referenced image not properly documented
4. **Command Output**: Missing actual command outputs and error messages
5. **Timeline**: No clear progression timeline or methodology explanation
6. **Root Access**: Writeup ends before achieving root - incomplete exploitation chain

#### Technical Improvements:

1. **Vulnerability Research**: Good identification of the Lambda layer vulnerability
2. **Docker Usage**: Proper containerization for exploit compatibility
3. **Credential Hygiene**: Smart filtering of competition environment vs. legitimate credentials
4. **Systematic Enumeration**: Good database discovery and analysis

#### Recommended Writeup Enhancements:

* Include nmap scan results
* Document web application enumeration methodology
* Add screenshots of key exploitation steps
* Show actual command outputs
* Complete the privilege escalation chain to root
* Include countermeasures and remediation advice

### Credentials Discovered

* `gael:mattp005numbertwo` (SSH)
* `backrest_root:!@#$%^` (Service account)



***

_Note: This writeup documents a controlled penetration testing exercise. All techniques should only be used in authorized testing environments._
