# FLAG AUTHORISER

## Intro

This is my write-up for the Web challenge "FLAG AUTHORISER" on the CTF site 247CTF.com.

## Challenge Details

The challenge involves finding a way to upgrade your access from an anonymous user to an admin.

## Steps to Solve

### 1. Access the Application

Start by accessing the web application provided for the challenge.

### 2. Analyze the Provided Code

```python

from flask import Flask, redirect, url_for, make_response, render_template, flash
from flask_jwt_extended import JWTManager, create_access_token, jwt_optional, get_jwt_identity
from secret import secret, admin_flag, jwt_secret

app = Flask(__name__)

cookie = "access_token_cookie"

app.config['SECRET_KEY'] = secret
app.config['JWT_SECRET_KEY'] = jwt_secret
app.config['JWT_TOKEN_LOCATION'] = ['cookies']
app.config['DEBUG'] = False

jwt = JWTManager(app)

def redirect_to_flag(msg):
    flash('%s' % msg, 'danger')
    return redirect(url_for('flag', _external=True))

@jwt.expired_token_loader
def my_expired_token_callback():
    return redirect_to_flag('Token expired')

@jwt.invalid_token_loader
def my_invalid_token_callback(callback):
    return redirect_to_flag(callback)

@jwt_optional
def get_flag():
    if get_jwt_identity() == 'admin':
        return admin_flag

@app.route('/flag')
def flag():
    response = make_response(render_template('main.html', flag=get_flag()))
    response.set_cookie(cookie, create_access_token(identity='anonymous'))
    return response

@app.route('/')
def source():
    return """
    %s
    """ % open(__file__).read()

if __name__ == "__main__":
    app.run()

```

Reviewing the code shows us that we are looking at a Flask web app that uses JWT (JSON Web Token). Visiting /flag will give us our token with identity='anonymous'. Our goal is to find how to change the anonymous JWT data to admin.

## Steps to Solve

### 1. Obtaining a Valid Cookie

Visit /flag and copy the access\_token, then visit https://jwt.io to see what data we got.

Let's change the identity to admin and replace the token with the new one.

Of course, it doesn't work 😂, so let's Google it.

We know that there's a secret key required for the token, which is why we got an error after trying the admin token.

### 2. Cracking the Token

With some Google searching, I found that cracking the secret key is possible using John the Ripper.

Check here for more info: https://www.freecodecamp.org/news/crack-passwords-using-john-the-ripper-pentesting-tutorial/

Place the token in a file and run this command (247CTF mentioned that we can use rockyou.txt as a wordlist to crack it faster instead of brute-forcing):

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt  # token stored in jwt.txt
```

\###3. Forging the Token Take the token, put it again in https://www.jwt.io.

We know the identity needs to be admin, so set that up in the parameters.

Add the secret.

Place the new token in the /flag route, and you'll get the flag.

### Resources

https://www.freecodecamp.org/news/crack-passwords-using-john-the-ripper-pentesting-tutorial/
