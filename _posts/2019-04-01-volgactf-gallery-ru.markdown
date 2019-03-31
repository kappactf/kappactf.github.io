---
layout: post
title:  "VolgaCTF 2019 Quals: web 300 — Gallery [RU]"
date:   2019-04-01 01:00:00 +0300
author: juwilie
excerpt: Райтап на таск Gallery с VolgaCTF 2019 Quals
---

> English version of this write-up will be published soon.

![Условие](/assets/img/2019/04/volgactf-gallery/statement.png){:class="preview-wide"}

Заходим на сайт, кроме формы входа ничего не видим:

![Форма входа](/assets/img/2019/04/volgactf-gallery/login.png){:class="preview-wide"}

Начнем решение с небольшого осмотра: запустим `dirsearh` на этом хосте. В выводе видим `package.json`, `sessions`, что намекает нам на то, что приложение написано на [node.js](https://nodejs.org/en/).

Со страницы входа находим JS-файл `js/main.js`, который обращается к API. Обнаруживаем, что на сервере подключен `autoindex` и мы можем посмотреть файлы в директории `js/`, где находим исходники серверной части приложения.

![index: js](/assets/img/2019/04/volgactf-gallery/js.png){:class="preview-wide"}

> [Сохраненная копия](/assets/files/2019/04/volgactf-gallery.zip) директории `js/` 

Выясняем, что приложение написано на фреймворке [Express](https://expressjs.com/) и доступно на порту 4000, куда запросы проксируются с порта 80 с помощью nginx. Имеет три метода: `/api/login` и `/api/logout` «не реализованы» и редиректят на `/login`, а последний проверяет сессию, и для юзера `admin` показывает флаг.

В `config.js` обнаруживаем, что приложение проксирует все неизвестные запросы на порт 5000 (правда, по всей видимости, этот порт не висит наружу), а также находим секретный ключ для подписи сессий. Выглядит как короткий путь к победе, однако, тут же выясняется, что сессии хранятся в файликах, файлик никак не создать, а существующих сессий в директории `sessions/` нет. Что ж, всё равно запомним.

![index: sessions](/assets/img/2019/04/volgactf-gallery/sessions.png){:class="preview-wide"}

На фронтэнде — в файле `main.js` — есть обращение к `/api/images`. Судя по всему, этот метод работает с файлами, что возможно поможет нам в решении таска. Обращаемся к данному методу, но получаем ошибку 403.

![/api/images: 403](/assets/img/2019/04/volgactf-gallery/images_403.png){:class="preview-wide"}

Возвращаясь к исходникам приложения, видим файл `auth.js`, в котором реализована проверка наличия сессии для всех путей, кроме `/api/login` и `/api/logout`. Также заметим, что `/api/images` не реализовано в Express-приложении, значит эта страница должна проксироваться на порт 5000. 

Пока мы не можем обойти данную проверку, попробуем понять, что же нас ждёт внутри: случайно решаем обратиться с методом `OPTIONS` на `/api/logout` — здесь авторизация не требуется, но и у Express такого метода нет, поэтому нас прокидывает внутрь, где мы видим ошибку PHP-фреймворка [Laravel](https://laravel.com/). 

![OPTIONS /api/login](/assets/img/2019/04/volgactf-gallery/options.png){:class="preview-wide"}

Тем временем вспоминаем или, как мы, случайно обнаруживаем, что Express не нормализует пути, и обходим авторизацию, используя URL `//api/images`. Получаем 200 и пустой список картинок. Далее вспоминаем параметр `year`. Понимаем, что нам дан функционал листинга директорий, и находим уязвимость Directory traversal.

![/api/images: 200](/assets/img/2019/04/volgactf-gallery/images_200.png){:class="preview-wide"}

![/api/images?year=2018](/assets/img/2019/04/volgactf-gallery/images_2018.png){:class="preview-wide"}

![/api/images?year=2018/..](/assets/img/2019/04/volgactf-gallery/images_dir_fail.png){:class="preview-wide"}

Благодаря включенному debug-режиму, понимаем, что после года подставляется `/img/`, что накладывает определенные ограничения на список доступных нам директорий. Однако, попробуем применить классический баг в PHP — Null byte injection, суть которого заключается в том, что функции на C воспринимают `%00` за конец строки. Это работает, и теперь мы видим директории без `/img/`, что даёт нам листинг всей файловой системы.

![Null byte injection](/assets/img/2019/04/volgactf-gallery/images_dir_null.png){:class="preview-wide"}

![/var/www](/assets/img/2019/04/volgactf-gallery/images_dir_www.png){:class="preview-wide"}

Искомый флаг, как мы могли понять и ранее, находится в `/var/www/flag`, однако ~~мы вам его не дадим~~ получить его с помощью метода `/api/image` невозможно, поскольку перед открытием файл передается в функцию `file_exists`, которая выдает ошибку при наличии `%00` в строке. Расстраиваемся и продолжаем искать.

![/api/image: fail #1](/assets/img/2019/04/volgactf-gallery/image_fail1.png){:class="preview-wide"}

![/api/image: fail #2](/assets/img/2019/04/volgactf-gallery/image_fail2.png){:class="preview-wide"}

![/api/image: fail #3](/assets/img/2019/04/volgactf-gallery/image_fail3.png){:class="preview-wide"}

![/api/image: fail #4](/assets/img/2019/04/volgactf-gallery/image_fail4.png){:class="preview-wide"}

Замечаем в директории `/var/www/apps` три приложения. С двумя из них мы уже знакомы: `volga_gallery` на PHP, а `volga_auth` на JS. А вот третье приложение под названием `volga_adminpanel` нам незнакомо.

![Applications](/assets/img/2019/04/volgactf-gallery/images_dir_apps.png){:class="preview-wide"}

![volga\_gallery](/assets/img/2019/04/volgactf-gallery/images_dir_gallery.png){:class="preview-wide"}

![volga\_auth](/assets/img/2019/04/volgactf-gallery/images_dir_apps.png){:class="preview-wide"}

![volga\_adminpanel](/assets/img/2019/04/volgactf-gallery/images_dir_adminpanel.png){:class="preview-wide"}

В директории видим файл `app.js` и директорию `sessions`, которая, в отличие от такой же директории в `volga_auth`, содержит какую-то сессию. Очень подозрительно. Применив смекалочку, догадаемся, что в этой сессии может быть `name=admin`. Мы какое-то время пытались обнаружить, где же крутится этот самый `app.js`, надеясь в нём найти что-то ещё, но таск оказался не таким простым.

![adminpanel: sessions](/assets/img/2019/04/volgactf-gallery/adminpanel_sessions.png){:class="preview-wide"}

Поскольку все пакеты в `npm` попадают только после тщательнейшего анализа ведущими специалистами по информационной безопасности, количество зеродеев в модулях лишь немногим больше бесконечности. Вспоминаем, что сессии хранятся в самой лучшей базе данных — в JSON-файликах. Всё это происходит с помощью модуля [session-file-store](https://www.npmjs.com/package/session-file-store). Потратив немного времени, находим [функцию](https://github.com/valery-barysok/session-file-store/blob/master/lib/session-file-helpers.js#L21), с помощью которой из имени куки составляется имя JSON-файла. Казалось бы, что может быть более безопасным. Однако, как вы уже догадываетесь, `path.join` просто сконкатенирует строки, вставив слеш между ними.

Внимательный читатель вспомнит, что сессий-то у нас нет, и в силу того, что сессии подписываются *секретным* ключом, ничего страшного подставить мы не сможем. Однако, сессия-то у нас есть — лежит она в файлике `sessions/../../volga_adminpanel/sessions/euzb7bMKx-5F29b2xNobGTDoWXmVFlEM.json` относительно корня приложения `volga_auth`. Чтобы открыть этот файлик, идентификатор сессии должен быть равен `../../volga_adminpanel/sessions/euzb7bMKx-5F29b2xNobGTDoWXmVFlEM`. Кроме этого, вспоминаем, что и секретный ключ у нас тоже есть, а значит можно идти и подписывать.

Раскуривать, как работает подпись сессии в Express нам не очень-то хотелось, поэтому мы применили грязный хак — подняли то же самое приложение у себя, добавили прямо в модуль `express-session` подписание нужной нам сессии, и ещё один метод API, который дергает создание сессии:

![Подписываем сессию](/assets/img/2019/04/volgactf-gallery/fakesession.png){:class="preview-wide"}

В ответ получили куку, с которой можно попробовать достать флаг методом `/api/flag`.

![Успех](/assets/img/2019/04/volgactf-gallery/final.png){:class="preview-wide"}

:triangular_flag_on_post: **VolgaCTF{31c2ac53d4101a01264775328797d424}**
