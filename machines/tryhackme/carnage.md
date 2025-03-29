# Carnage

## PCAP Analysis Write-up

### Scenario

Eric Fischer from the Purchasing Department at Bartell Ltd received an email from a known contact containing a Word document. Upon opening the document and clicking "Enable Content," he unknowingly allowed macros to execute. Shortly after, the Security Operations Center (SOC) team detected suspicious outbound connections from his workstation. A network capture (PCAP) file was collected for analysis, and our task is to investigate it.

***

### Task 1: Starting the Investigation

To begin, we need to analyze the provided PCAP file. The virtual machine in the TryHackMe (THM) room is preconfigured with the necessary tools, though it may be slow.

***

### Task 2: Traffic Analysis

We open **Wireshark** and load the file `carnage.pcap` from the `Analysis` folder on the desktop.

#### **Question 1: What was the date and time for the first HTTP connection to the malicious IP?**

**Approach:**

1.  Apply an HTTP filter in Wireshark:

    ```
    http
    ```
2. Locate the first **GET** request directed to the suspicious destination IP.
3. Convert the timestamp format by navigating to:
   * **View → Time Display Format**

The exact request can be identified based on the first instance of HTTP communication with the malicious IP.

#### **Question 2: What is the name of the ZIP file that was downloaded?**

**Approach:**

1. Locate the first **GET** request.
2. Right-click the request and select **Follow → HTTP Stream**.
3. The ZIP file's name appears within the request URL.

#### **Question 3: What was the domain hosting the malicious ZIP file?**

**Approach:**

The domain name can be found in the same HTTP request where the ZIP file was downloaded. The **Host** header contains the domain hosting the malicious file.

#### **Question 4: What is the name of the file inside the ZIP file?**

**Approach:**

1. Locate an HTTP **200 OK** response for the ZIP download.
2. Identify the filename from the response.
3. Alternatively, extract the file:
   * Click **File** (top-left corner of Wireshark).
   * Select **Export Objects → HTTP**.
   * Locate and extract the malicious ZIP file.
   * Unzip it to reveal its contents.

#### **Question 5: What is the name of the web server?**

**Approach:**

1. Follow the HTTP stream of the request.
2. Look for the **Server** header in the HTTP response.

#### **Question 6: What is the version of the web server?**

**Approach:**

The version information is typically found in the **X-Powered-By** header within the HTTP response.

***

### Finding Malicious Domains

To uncover additional domains involved in the attack, we analyze HTTPS (SSL/TLS) traffic.

#### **Approach:**

1. Identify the relevant timeframe (16:45:11 - 16:45:30).
2. Navigate to **Edit → Preferences → Name Resolution**, and enable "Resolve Network IP Addresses."
3.  Apply the following Wireshark filter to isolate TLS traffic:

    ```
    tls.handshake.type == 1 and (frame.time >= "YYYY-MM-DD HH:MM:SS") && (frame.time <= "YYYY-MM-DD HH:MM:SS")
    ```

#### **Question 7: What were the three domains involved in malicious activity?**

* Analyze the filtered packets.
* Cross-check the domains on **VirusTotal** to confirm malicious activity.

#### **Question 8: Which certificate authority issued the SSL certificate to the first domain?**

**Approach:**

1. Examine the TLS handshake packets.
2. Look at the **Certificate** details to find the issuing authority.

***

### Identifying Cobalt Strike C2 Servers

Cobalt Strike is commonly used for command and control (C2) operations.

#### **Approach:**

1.  Apply a filter to locate **GET** requests:

    ```
    http.request.method == "GET"
    ```
2. Check **Statistics → Conversations** to identify IPs with repeated connections.
3. Verify suspicious IPs using **VirusTotal (Community Tab)** to determine if they belong to known Cobalt Strike C2 servers.

#### **Question 9: What are the two IPs of the Cobalt Strike servers?**

* Identified through the above analysis.

#### **Question 10: What is the Host header for the first Cobalt Strike IP?**

* Located in the **Community Tab** of VirusTotal or the HTTP stream in Wireshark.

#### **Question 11 & 12: What are the domain names of the Cobalt Strike servers?**

* Found in **VirusTotal** or **Wireshark HTTP streams**.

#### **Question 13: What is the domain name of the post-infection traffic?**

*   Apply a filter to examine **POST** requests:

    ```
    http.request.method == "POST"
    ```
* Follow the TCP stream to analyze the domain used for data exfiltration.

#### **Question 14: What are the first eleven characters that the victim host sends out to the malicious domain?**

* Observed in the same TCP stream as the **POST** request.

#### **Question 15: What was the length of the first packet sent to the C2 server?**

* Packet length is visible under the **Length** tab in Wireshark.

#### **Question 16: What was the Server header for the malicious domain?**

* Located in the TCP stream of the malicious request.
*   Use:

    ```
    frame contains "api"
    ```

#### **Question 17: When did the DNS query for the victim's IP check occur?**

**Approach:**

1.  Filter for DNS queries containing "api":

    ```
    dns && frame contains "api"
    ```
2. Identify the relevant timestamp in packet 24,147.

#### **Question 18: What was the domain in the DNS query?**

* Found in the **UDP (DNS) stream**.

#### **Question 19: What was the first MAIL FROM address observed in the traffic?**

**Approach:**

1.  Filter for SMTP-related traffic:

    ```
    frame contains "MAIL FROM"
    ```
2. Identify the first sender address in the TCP stream.

#### **Question 20: How many packets were observed for SMTP traffic?**

* Apply an **SMTP filter** and check the total packet count at the bottom of Wireshark.

***

### Conclusion

By analyzing the PCAP file, we uncovered:

1. The initial infection via a malicious macro-enabled Word document.
2. A ZIP file download from a suspicious domain.
3. Cobalt Strike C2 infrastructure used for persistence and communication.

Throughout this process, we leveraged packet filtering, TCP stream analysis, and OSINT tools like VirusTotal to track the attack. This analysis highlights the importance of network forensics in identifying and mitigating threats. Great job!
