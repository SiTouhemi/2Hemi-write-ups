# Insomnia

### **Step-by-Step Walkthrough**

**Phase 1: Recon**

1. First, explore the website thoroughly. Visit every endpoint to understand its functionality before diving into the source code.
2. Nothing intriguing? Time to peek behind the curtains—check out the source code, where two interesting snippets await. 😏

***

**Phase 2: Understanding the Code**

Here's the juicy part :

```java
class ProfileController extends BaseController
{
    public function index()
    {
        $token = (string) $_COOKIE["token"] ?? null;
        $flag = file_get_contents(APPPATH . "/../flag.txt");
        if (isset($token)) {
            $key = (string) getenv("JWT_SECRET");
            $jwt_decode = JWT::decode($token, new Key($key, "HS256"));
            $username = $jwt_decode->username;
            if ($username == "administrator") {
                return view("ProfilePage", [
                    "username" => $username,
                    "content" => $flag,
                ]);
            } else {
                $content = "Haven't seen you for a while";
                return view("ProfilePage", [
                    "username" => $username,
                    "content" => $content,
                ]);
            }
        }
    }
}
```

To grab the flag, you need to impersonate the **administrator**. since the JWT token has a random secret each time it’s built and is properly verified, I ruled out the possibility of a session management vulnerability. Instead, I focused on the login function to see what it offers..

```java
 public function login()
    {
        $db = db_connect();
        $json_data = request()->getJSON(true);
        --------------------------------------------------------------------------
        if (!count($json_data) == 2) {
            return $this->respond("Please provide username and password", 404);
        }
        --------------------------------------------------------------------------
        $query = $db->table("users")->getWhere($json_data, 1, 0);
        $result = $query->getRowArray();
        if (!$result) {
            return $this->respond("User not found", 404);
        } else {
            $key = (string) getenv("JWT_SECRET");
            $iat = time();
            $exp = $iat + 36000;
            $headers = [
                "alg" => "HS256",
                "typ" => "JWT",
            ];
            $payload = [
                "iat" => $iat,
                "exp" => $exp,
                "username" => $result["username"],
            ];
            $token = JWT::encode($payload, $key, "HS256");
            $response = [
                "message" => "Login Succesful",
                "token" => $token,
            ];
            return $this->respond($response, 200);
        }
    }
```

It's hard to spot, I know, but let's dive into analyzing it **The Vulnerability**

```java
if (!count($json_data) == 2) {
            return $this->respond("Please provide username and password", 404);
        }
```

The `!` operator is applied directly to the `count()` result. This flips the numeric result into a boolean:

* `count($json_data)` returns the number of keys in the JSON object.
* If `count($json_data)` equals `2`:
  * `!2` evaluates to `false`.
  * `false == 2` is always `false`.

So, this condition **never blocks invalid input**, regardless of what you send.

As a result, the condition `if (!count($json_data) == 2)` evaluates to `false`, preventing the `return` statement from being executed. This allows the program to continue processing the login.

Additionally, attempting to log in with only a username won’t work because the password field is still sent, even if left empty.

![](<../../.gitbook/assets/image (9).png>)

## **Exploitation**

Our goal is to log in as the **administrator** without knowing the password. Here’s how:

1. Intercept the request in Burp Suite or a similar tool.
2. Remove the `password` key entirely. This bypasses the check because the flawed validation logic lets any input through.

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

It worked :)

**Phase 3: Harvesting the Flag**

1. Log in with your normal account.
2. Open your browser's developer tools (`Inspect Element`).
3. Navigate to **Application > Cookies**.
4. Replace your token with the stolen **administrator** JWT.
5. Refresh the page. Voilà—the flag is yours. 🏴‍☠️
