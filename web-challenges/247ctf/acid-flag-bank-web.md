# ACID FLAG BANK (web)

### Intro

This is my write-up for the web challenge "ACID FLAG BANK" on the CTF site 247CTF.com.

### Challenge Details

The challenge involves finding a way to increase the total available funds and buy a flag.

### Steps to Solve

### 1. Access the Application

Start by accessing the web application provided for the challenge.

### 2. Analyze the Provided Code

```python

<?php
require_once('flag.php');

class ChallDB
{
    public function __construct($flag)
    {
        $this->pdo = new SQLite3('/tmp/users.db');
        $this->flag = $flag;
    }
 
    public function updateFunds($id, $funds)
    {
        $stmt = $this->pdo->prepare('update users set funds = :funds where id = :id');
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $stmt->bindValue(':funds', $funds, SQLITE3_INTEGER);
        return $stmt->execute();
    }

    public function resetFunds()
    {
        $this->updateFunds(1, 247);
        $this->updateFunds(2, 0);
        return "Funds updated!";
    }

    public function getFunds($id)
    {
        $stmt = $this->pdo->prepare('select funds from users where id = :id');
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        return $result->fetchArray(SQLITE3_ASSOC)['funds'];
    }

    public function validUser($id)
    {
        $stmt = $this->pdo->prepare('select count(*) as valid from users where id = :id');
        $stmt->bindValue(':id', $id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray(SQLITE3_ASSOC);
        return $row['valid'] == true;
    }

    public function dumpUsers()
    {
        $result = $this->pdo->query("select id, funds from users");
        echo "<pre>";
        echo "ID FUNDS\n";
        while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
            echo "{$row['id']}  {$row['funds']}\n";
        }
        echo "</pre>";
    }

    public function buyFlag($id)
    {
        if ($this->validUser($id) && $this->getFunds($id) > 247) {
            return $this->flag;
        } else {
            return "Insufficient funds!";
        }
    }

    public function clean($x)
    {
        return round((int)trim($x));
    }
}

$db = new ChallDB($flag);
if (isset($_GET['dump'])) {
    $db->dumpUsers();
} elseif (isset($_GET['reset'])) {
    echo $db->resetFunds();
} elseif (isset($_GET['flag'], $_GET['from'])) {
    $from = $db->clean($_GET['from']);
    echo $db->buyFlag($from);
} elseif (isset($_GET['to'], $_GET['from'], $_GET['amount'])) {
    $to = $db->clean($_GET['to']);
    $from = $db->clean($_GET['from']);
    $amount = $db->clean($_GET['amount']);
    if ($to !== $from && $amount > 0 && $amount <= 247 && $db->validUser($to) && $db->validUser($from) && $db->getFunds($from) >= $amount) {
        $db->updateFunds($from, $db->getFunds($from) - $amount);
        $db->updateFunds($to, $db->getFunds($to) + $amount);
        echo "Funds transferred!";
    } else {
        echo "Invalid transfer request!";
    }
} else {
    echo highlight_file(__FILE__, true);
}
```

The PHP script manages user funds in an SQLite database, allowing fund transfers, resets, and flag retrieval based on user validation. Users with more than 247 funds can buy the flag, while fund transfers must adhere to specified conditions. It includes functions to dump user data, validate users, and clean input values.

### Steps to Solve

### 1. Understanding and Identifying the Vulnerability

Let's try to transfer funds from ID 1 to ID 2:

```vbnet
https://f8011c2afbe1a1be.247ctf.com/?to=2&from=1&amount=247
```

Funds transferred!

Let's check if the funds have really been transferred:

http://yourserver.com/yourfile.php?dump

Yes, we see 247 in ID 2 and 0 in ID 1.

We can see that there is a lot of processing happening before the transfer occurs:&#x20;

<figure><img src="../../.gitbook/assets/image1.png" alt=""><figcaption></figcaption></figure>

This indicates that there is a race condition vulnerability in the transfer function.

### Explanation

A race condition vulnerability occurs when the outcome of a process depends on the timing or order of events, often because the code does not properly synchronize access to shared resources or performs checks only after executing actions.

You can read more about this vulnerability at PortSwigger's guide on race conditions.

### 2. Exploiting the Vulnerability

Theoretically, if we send multiple requests almost simultaneously to transfer money between accounts, the checks for both requests might be processed at the same time. As a result, both requests could pass the checks and execute the transactions concurrently.

First, open Burp Suite and capture the transfer funds request:&#x20;

<figure><img src="../../.gitbook/assets/image2.png" alt=""><figcaption></figcaption></figure>

Second, send it to Repeater. Right-click on the request, then go to the "Tab" group and duplicate it many times.

Change the "Send" button to "Send group (parallel)":

![Image 3](../../.gitbook/assets/image3.png)

I got a 200 response in only one request. Let's reset the funds and try again with more duplicated requests:

![Image 5](../../.gitbook/assets/image5.png)

Now we get a 200 response in many tabs.

Check the funds with ?dump:

Finally, it worked, and we have more than 247 funds :). We can now buy the flag using:

```csharp
?flag=1&from=2
```
