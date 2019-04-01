---
layout: post
title:  "VolgaCTF 2019 Quals: web 300 — Gallery [EN]"
date:   2019-04-01 01:00:00 +0300
author: juwilie
excerpt: Write-up for task Gallery from VolgaCTF 2019 Quals
hidden: true
---

> Russian version can be found [here](/volgactf-gallery-ru/)

![Task](/assets/img/2019/04/volgactf-gallery/statement.png){:class="preview-wide"}

Let's open the website. We can see only login form here.

![Login form](/assets/img/2019/04/volgactf-gallery/login.png){:class="preview-wide"}

We've run `dirsearch` on this host and found names like `package.json` and `sessions`. It probably means that it's a [node.js](https://nodejs.org/en/) application. Also we can find file `js/main.js`, which makes requests to API. We noticed that `autoindex` was enabled and we're able to see the whole `js/` directory. And there is server side sources.

![index: js](/assets/img/2019/04/volgactf-gallery/js.png){:class="preview-wide"}

> There is [saved copy](/assets/files/2019/04/volgactf-gallery.zip) of `js/` directory

Application is written on [Express](https://expressjs.com/) framework and served on port 4000. There is nginx on port 80, which forwards requests to Express. Application has only three methods: `/api/login` and `/api/logout` are not implemented. They return redirects to `/login`. Last endpoint checks your session and shows flag only if your name is `admin`.

It turns out that all unknown requests are forwarded to port 5000 from `config.js` file. But this port isn't available from outside. Also we found secret key which is used in session signing. It seemed that task will be easy, but sessions are stored in files, not in cookies. We can't create new session, and there is no sessions in the respective directory. So we'll just remember it, right?

![index: sessions](/assets/img/2019/04/volgactf-gallery/sessions.png){:class="preview-wide"}

There is frontend script `js/main.js`. And there is request to `/api/images`. We can guess that this method works with files and it may help us. But we get error 403 when requesting the page.

![/api/images: 403](/assets/img/2019/04/volgactf-gallery/images_403.png){:class="preview-wide"}

Let's get back to server sources. There is session check in `auth.js` file, so we can access only `/api/login` and `/api/logout`. Also we noted that there is no `/api/images` implementation in Express application, so it may be implemented in the hidden application on port 5000.

We're not able to pass this check at the moment, but let's try to find out something about that hidden app. We discovered by accident that if we send `OPTIONS` request to `/api/logout`, we'll get [Laravel](https://laravel.com/) error page. It's PHP framework. But why? It's obvious: we aren't required to log in because path is whitelisted, but there is no Express endpoint for this path and method.

![OPTIONS /api/login](/assets/img/2019/04/volgactf-gallery/options.png){:class="preview-wide"}

Then we can discover or remember that Express doesn't normalize paths. Therefore authorization bypass is easy: we just need to follow URL `//api/images`. We got `200 OK` there and empty list. If we substitute `year` parameter, we'll get image list. So we have directories listing and it's vulnerable to Directory traversal.

![/api/images: 200](/assets/img/2019/04/volgactf-gallery/images_200.png){:class="preview-wide"}

![/api/images?year=2018](/assets/img/2019/04/volgactf-gallery/images_2018.png){:class="preview-wide"}

![/api/images?year=2018/..](/assets/img/2019/04/volgactf-gallery/images_dir_fail.png){:class="preview-wide"}

The debug mode was switched on and it helped us to understand that there is `/img/` after year. So we can't see arbitrary directory contents. But you may remember about classic PHP bug — null byte injection: C functions recognize `%00` as the end of string. It helps us to avoid `/img/` ending and to list any directory in the filesystem.

![Null byte injection](/assets/img/2019/04/volgactf-gallery/images_dir_null.png){:class="preview-wide"}

![/var/www](/assets/img/2019/04/volgactf-gallery/images_dir_www.png){:class="preview-wide"}

Flag is located in `/var/www/flag`, but we can't access it using `/api/image` because of `file_exists` call: it fails when there is `%00` in string. Also there is filtering in `img` parameter, so nothing helps.

![/api/image: fail #1](/assets/img/2019/04/volgactf-gallery/image_fail1.png){:class="preview-wide"}

![/api/image: fail #2](/assets/img/2019/04/volgactf-gallery/image_fail2.png){:class="preview-wide"}

![/api/image: fail #3](/assets/img/2019/04/volgactf-gallery/image_fail3.png){:class="preview-wide"}

![/api/image: fail #4](/assets/img/2019/04/volgactf-gallery/image_fail4.png){:class="preview-wide"}

Then we noted three applications located in `/var/www/apps`. We know two of them: `volga_gallery` is Laravel app and `volga_auth` is Express app. But we don't know anything about `volga_adminpanel`.

![Applications](/assets/img/2019/04/volgactf-gallery/images_dir_apps.png){:class="preview-wide"}

![volga\_gallery](/assets/img/2019/04/volgactf-gallery/images_dir_gallery.png){:class="preview-wide"}

![volga\_auth](/assets/img/2019/04/volgactf-gallery/images_dir_apps.png){:class="preview-wide"}

![volga\_adminpanel](/assets/img/2019/04/volgactf-gallery/images_dir_adminpanel.png){:class="preview-wide"}

We see `app.js` and `sessions` here, but this directory contains one session. It's very suspicious. We thought about it and guessed that this session may contain `name=admin` property. Then we started to search this “admin panel”, but we failed. In fact it's a bit harder.

![adminpanel: sessions](/assets/img/2019/04/volgactf-gallery/adminpanel_sessions.png){:class="preview-wide"}

As you know, letter **S** in `npm` stands for security. So we remembered that sessions are stored inside JSON files. This functionality is provided by [session-file-store](https://www.npmjs.com/package/session-file-store) module. We've got into this library and [found](https://github.com/valery-barysok/session-file-store/blob/master/lib/session-file-helpers.js#L21) how session filename is generated. It's just prefix (`sessions/` in our case) concatenated with session ID from cookie with a slash.

But we don't have sessions and they're signed with *secret* key, so we can't put anything malicious here. However we have one: what will happen if we substitute `../../volga_adminpanel/sessions/euzb7bMKx-5F29b2xNobGTDoWXmVFlEM` as session ID? As you may guess, Express will find the session. And we have secret key, so we can sign a session.

We didn't want to explore how session signing works in Express, so we ran copy of application locally, added session signing right inside `express-session` module and triggered it with an additional endpoint:

![Session signing](/assets/img/2019/04/volgactf-gallery/fakesession.png){:class="preview-wide"}

We got a cookie so let's get `/api/flag`:

![Success](/assets/img/2019/04/volgactf-gallery/final.png){:class="preview-wide"}

:triangular_flag_on_post: **VolgaCTF{31c2ac53d4101a01264775328797d424}**
