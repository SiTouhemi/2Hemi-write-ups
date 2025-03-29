# Secured Session

## Intro

This is my write-up for the Web challenge "Secured Session" on the CTF site [247CTF.com](https://247ctf.com/).

## Challenge Details

The challenge involves guessing a random secret key to retrieve a flag securely stored in your session.

## Steps to Solve

### 1. Access the Application

Start by accessing the web application provided for the challenge.

### 2. Analyze the Provided Python Code

The provided Python Flask code is as follows:

import os from flask import Flask, request, session from flag import flag

app = Flask(**name**) app.config\['SECRET\_KEY'] = os.urandom(24)

def secret\_key\_to\_int(s): try: secret\_key = int(s) except ValueError: secret\_key = 0 return secret\_key

@app.route("/flag") def index(): secret\_key = secret\_key\_to\_int(request.args\['secret\_key']) if 'secret\_key' in request.args else None session\['flag'] = flag if secret\_key == app.config\['SECRET\_KEY']: return session\['flag'] else: return "Incorrect secret key!"

@app.route('/') def source(): return

if **name** == "**main**": app.run()

Analyzing the code, we see it involves cookies and a random key. Let’s check our cookies:

![](<../../.gitbook/assets/image1 (1).png>)

3. Check the Provided Information Notice that there are no session cookies. Let's check /flag:

![](<../../.gitbook/assets/image2 (1).png>)

The message indicates an incorrect key. Since the key is stored in the session cookie, let's analyze it.

After researching Flask sessions, I found this website that explains how Flask cookies work. I realized that Flask cookies are similar to JSON Web Tokens (JWT), where data is separated by dots (.).

I used jwt.io to analyze the JWT cookie:

![](<../../.gitbook/assets/image3 (1).png>)

By pasting the cookie into the JWT.io decoder, you can see its content, which includes the flag data, base64 encoded.

Decode the base64 string using the following command:

$ echo "MjQ3Q1RGe2RhODA3OTVmOGE1Y2FiMmUwMzdkNzM4NTgwN2I5YTkxfQ==" | base64 -d The decoded output will be the flag: 247CTF{xxxxxxxx}
