---
layout: post
title: Wacon2023 WriteUp(Web)
date: 2023-09-08 00:01 +0900
tags: [ctf, web, writeup, wacon]
categories: [ctf, writeup]
toc: true
---

# Wacon2023

## Mosaic

### Recon

First thing you see are a login and register button

![Untitled](/assets/img/20230908/Untitled.png)

After you register and login you see 3 menus

![Untitled](/assets/img/20230908/Untitled%201.png)

1. `Upload` : Can Upload .jpg, .zip, .tiff, .png files. After uploading it shows the path to the uploaded file
2. `Mosaic` : Able to see an uploaded file mosaicked

The files are as followed (excluding asdasd.png)

![Untitled](/assets/img/20230908/Untitled%202.png)

The weird this is that the `password.txt` file was given. In most CTFs the password is within the source code.

### Objective

The flag is placed as followed

![Untitled](/assets/img/20230908/Untitled%203.png)

- At the root directory the image file is saved and the path is saved in the `FLAG` variable

![Untitled](/assets/img/20230908/Untitled%204.png)

- The flag image will be copied by the `copyfile` function, to a possibly accessible directory only if the session is username is admin(== admin session) and it is called locally (127.0.0.1)

![Untitled](/assets/img/20230908/Untitled%205.png)

- Lastly, by using `/check_upload@<username>/<file>` , you are able to see an image but to see an image in admin, you must have a session with username “admin”

### Step 1. Admin Password

Of course you need the admin password to login as admin. The password for admin is saved in `password.txt`

![Untitled](/assets/img/20230908/Untitled%206.png)

- FYI the password.txt is saved at `/app/password.txt`

To read the password you can use the code below.

![Untitled](/assets/img/20230908/Untitled%207.png)

The image file you want to read depends on the `username` name you input and you cannot use `../` in the `file` path

But, you can read the `password.txt` by placing `..` in the `username` and it will form `/app/uploads/../password.txt`

![Untitled](/assets/img/20230908/Untitled%208.png)

### Step 2. Copy Flag Image

The next step is accessing the root URL locally (127.0.0.1) with the admin session.

![Untitled](/assets/img/20230908/Untitled%209.png)

As you can see in the code, depending on the `image_url` value it is used differently. If we were to use `requests.get` to call 127.0.0.1 we are able to copy the flag image.

But `image_url` filters values that start with `https://` and `http://`

![Untitled](/assets/img/20230908/Untitled%2010.png)

You can bypass this by using capital letters or using a python vuln like below.

- 참고 : [https://thehackernews.com/2023/08/new-python-url-parsing-flaw-enables.html](https://thehackernews.com/2023/08/new-python-url-parsing-flaw-enables.html)

The value goes straight through with the space in front of http

![Untitled](/assets/img/20230908/Untitled%2011.png)

It might seem that the logic will work by putting 127.0.0.1 in the `image_url` parameter but you will get 500 error

![Untitled](/assets/img/20230908/Untitled%2012.png)

![Untitled](/assets/img/20230908/Untitled%2013.png)

Like you can see the error above an error occurs in the `split` function because there is no value for the mimetype.

![Untitled](/assets/img/20230908/Untitled%2014.png)

The `guess_type` function in `mimetype` tries to guess the mimetype of a uri depending on the extension(after the ‘`.`’)

To bypass this you can do it as below

![Untitled](/assets/img/20230908/Untitled%2015.png)

Again you can see a 500 error

![Untitled](/assets/img/20230908/Untitled%2016.png)

But this time it occurs in the `imageio.imread` function and if you look at the log, the request locally worked

![Untitled](/assets/img/20230908/Untitled%2017.png)

Thus you can get the flag image by accessing `/check_upload/@admin/flag.png`

![Untitled](/assets/img/20230908/Untitled%2018.png)

## Warmup-Revenge

### Recon

When access the web page you get to see a page like below

![Untitled](/assets/img/20230908/Untitled%2019.png)

After you register and login you will see as below

![Untitled](/assets/img/20230908/Untitled%2020.png)

1. in the board menu, you can see peoples titles, writers and date of writings and you can write one yourself

![Untitled](/assets/img/20230908/Untitled%2021.png)

![Untitled](/assets/img/20230908/Untitled%2022.png)

![Untitled](/assets/img/20230908/Untitled%2023.png)

1. In the Myinfo page, you can look at infos of yourself and edit it.
    
    ![Untitled](/assets/img/20230908/Untitled%2024.png)
    
2. Note is a function to send and receive DMs
    
    ![Untitled](/assets/img/20230908/Untitled%2025.png)
    

The environment is as follows

![Untitled](/assets/img/20230908/Untitled%2026.png)

### Objective

The main goal is to achieve the admin’s cookie which is the “flag” which means there is a 95% chance that it will be an XSS challenge

![bot.js](/assets/img/20230908/Untitled%2027.png)

bot.js

The only endpoint where the “User” and “Admin” interact is `/report.php`

![Untitled](/assets/img/20230908/Untitled%2028.png)

The user can send the `path` and `idx` of an article to the admin

- FYI, everything you write in text will be filtered.

![Untitled](/assets/img/20230908/Untitled%2029.png)

### Step 1. XSS Entry Point

While looking around where I might be able to enter by XSS script, I found a download logic which looked strange.

![Untitled](/assets/img/20230908/Untitled%2030.png)

The `content-disposition` header is setted differently depending on the `user-agent`, but since you cannot change the admin bot’s UA I didn’t think too big about this.

I thought if I were to delete the header or possibly change(inject) it, I might be able to make a URL that won’t popup a download screen.

![Untitled](/assets/img/20230908/Untitled%2031.png)

![Untitled](/assets/img/20230908/Untitled%2032.png)

Trying to figure out how, I did a bit of fuzzing and I was able to remove the `content-disposition` header in my response. (The reason in the link below)

- 참고 : [https://github.com/php/php-src/blob/master/main/SAPI.c#L739](https://github.com/php/php-src/blob/master/main/SAPI.c#L739)

When I put in a “hex” value of `\r` in `filename` parameter, it will delete the `content-disposition` response header due to an error

![Untitled](/assets/img/20230908/Untitled%2033.png)

![Untitled](/assets/img/20230908/Untitled%2034.png)

### Step 2. Bypass CSP

As you can see in the image above, there is a CSP so I cannot not use script right off the bat.

```
Content-Security-Policy: default-src 'self'; style-src 'self' https://stackpath.bootstrapcdn.com 'unsafe-inline'; script-src 'self'; img-src data:
```

To bypass it, I thought “why not just upload a `js` file and make `script src` call it?”

![Untitled](/assets/img/20230908/Untitled%2035.png)

![Untitled](/assets/img/20230908/Untitled%2036.png)

![Untitled](/assets/img/20230908/Untitled%2037.png)

Thus I was able to make the bot send me his cookie by crafting a payload like below.

![Untitled](/assets/img/20230908/Untitled%2038.png)

![Untitled](/assets/img/20230908/Untitled%2039.png)

![Untitled](/assets/img/20230908/Untitled%2040.png)

![Untitled](/assets/img/20230908/Untitled%2041.png)