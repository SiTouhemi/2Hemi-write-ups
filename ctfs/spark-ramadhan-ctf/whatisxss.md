# WhatIsXSS

<figure><img src="https://miro.medium.com/v2/resize:fit:550/1*SQiVwympl3hvplsAk941gA.png" alt="" height="438" width="489"><figcaption></figcaption></figure>

Upon opening the web application, we find a simple page explaining the XSS vulnerability and how it works.

Inspecting the source code, we discover a `script.js` file that contains the flag. However, the script is obfuscated, so we need to perform JavaScript deobfuscation.

Within the script, we encounter multiple fake flags, but the correct flag is stored inside a function called `revealFlagyabro()`.

By executing a standard **Stored XSS** payload

```
<img src=x onerror=revealFlagyabro() >
```

we successfully capture the flag, which is base64-encoded:\
Spark{Y0u\_N33d\_t0\_l34Rn\_XSS!!!!!}
