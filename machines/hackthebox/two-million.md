# Two Million

Js beatifier

```
https://js-beautify.com
```

Submit js function :

!\[\[Pasted image 20241222175016.png]]

***

**API**

“Connection Pack” sends a GET request to `/api/v1/user/vpn/generate`, and “Regenerate” sends a GET to `/api/v1/user/vpn/regenerate`.

lets check /api/v1

***

If the server is doing something like `gen_vpn.sh [username]`, then letsl try putting a `;` in the username Whoah ! its Command Injection :) SO guessy huh ?
