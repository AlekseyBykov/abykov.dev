---
title: "Развертывание блога на GitHub Pages с темой Chirpy"
layout: post
date: 2025-04-30 12:00:00 +0300
categories: [GitHub, Блог]
tags: [jekyll, github-pages, chirpy, ruby, deploy, dns, ssl, github-actions]
---

В этой статье рассмотрим, как развернуть персональный блог, например `abykov.dev`, на GitHub Pages с использованием
Jekyll темы. В качестве основы возьмем тему **Chirpy** для Jekyll, которая, на мой взгляд, отлично подходит для блога.

**Chirpy** — мощная и современная тема Jekyll с минималистичным дизайном, встроенным поиском, поддержкой категорий, 
тегов, комментариев, RSS и даже MathJax для формул.

Пошагово разберем, как клонировать тему, адаптировать ее под себя и опубликовать сайт.

> ⚠️ В этой статье мы клонируем саму тему, чтобы иметь полный контроль над шаблонами и стилями. 
> Это удобно для изучения и кастомизации, но обновление версии темы необходимо выполнять вручную. 
> В целом, рекомендуют использовать стартер-проект [Chirpy Starter](https://github.com/cotes2020/chirpy-starter), 
> который подключает тему как Ruby-гем.

## Настройки на стороне хостинг-провайдера

Для начала зарегистрируем домен `abykov.dev` у хостинг-провайдера, например на [reg.ru](https://www.reg.ru),
которому делегируем DNS resolve. Через этот же провайдер настроим SSL-сертификат и обеспечим редирект 
с `www.abykov.dev` на `abykov.dev`. При этом сайт будет хоститься на GitHub Pages.

> ⚠️ Почему это важно?
> GitHub Pages может обслуживать только **один кастомный домен** — либо `abykov.dev`, либо `www.abykov.dev`. 
> Одновременная поддержка обоих с валидным HTTPS-сертификатом недоступна.

Т.е. мы регистрируем только основной домен `abykov.dev`, далее настраиваем `www.abykov.dev` как поддомен 
(основной домен владеет всеми своими поддоменами). Основной домен будет указывать на GitHub Pages,
а поддомен - на сервера reg.ru. Запросы на поддомен будут переадресовываться на основной -
настраиваем веб-сервер так, чтобы он делал **301-редирект** на `https://abykov.dev`.
![Создание сайта www.abykov.dev](/assets/img/www-301-redirect.png){: .shadow .rounded }

Это позволяет использовать короткий и каноничный адрес без `www`, при этом сохранить работоспособность 
и узнаваемость адреса с `www`.

Выполним настройки веб-сервера и DNS на стороне reg.ru в ISPManager.

### Регистрация поддомена и редирект

В разделе Sites добавим поддомен `www.abykov.dev` и настроим редирект:

- IP-адрес: `37.140.192.242` (сервер reg.ru)
- Включен SSL (Let's Encrypt сертификат для `www.abykov.dev`)
- Включен редирект всех HTTP-запросов на HTTPS
- Веб-сервер делает редирект с `www.abykov.dev` на `https://abykov.dev`

![Создание сайта www.abykov.dev](/assets/img/www-site-setup.png){: .shadow .rounded }

Если хостинг-провайдер поддерживает управление через файл `.htaccess`, пропишем редирект вручную 
(если автоматический недоступен):

```text
RewriteEngine On
RewriteCond %{HTTP_HOST} ^www\.abykov\.dev$ [NC]
RewriteRule ^(.*)$ https://abykov.dev/$1 [L,R=301]
```

### Подключение SSL-сертификата
Для безопасного соединения настроим SSL-сертификат для поддомена `www.abykov.dev`. 
В панели управления reg.ru достаточно включить бесплатный сертификат Let's Encrypt, который автоматически обновляется.
Если автоматическая установка недоступна, можно создать сертификат вручную:
- Сгенерировать запрос на сертификат (CSR) через ISPManager.
- Выпустить бесплатный сертификат Let's Encrypt или приобрести коммерческий.
- Установить сертификат в разделе управления SSL.

![Создание сайта www.abykov.dev](/assets/img/www-ssl-cert.png){: .shadow .rounded }

> ⚠️ SSL необходим не только для шифрования, но и для того, чтобы браузеры не помечали сайт как "небезопасный". 
> Также это обязательное требование для GitHub Pages при работе с кастомными доменами.

Так мы выпустили сертификат для `www.abykov.dev` на стороне reg.ru. 
Это нужно потому, что reg.ru отвечает за этот поддомен и именно он обрабатывает HTTPS-запросы на `www.abykov.dev` до того, 
как сделать редирект на основной домен `abykov.dev`. Для основного домена `abykov.dev`, который обслуживается GitHub Pages, 
сертификат выпускает и обновляет сам GitHub.

### Настройка DNS

Для корректной работы сайта и почты добавим следующие DNS-записи:

```text
A  abykov.dev       → 185.199.108.153 (GitHub Pages)
A  www.abykov.dev   → 37.140.192.242 (сервер reg.ru)
```
Основной домен `abykov.dev` указывает на IP GitHub Pages, что позволяет обслуживать сайт напрямую с GitHub Pages.
Поддомен `www.abykov.dev` указывает на IP reg.ru, где настроен редирект на основной домен.

Дополнительно остаются стандартные записи:

- MX-записи — для работы почты (направляют почтовый трафик на почтовые серверы reg.ru)
- TXT-записи SPF и DKIM — используются для email-аутентификации. Помогают предотвратить спам и подтверждают подлинность исходящей почты

![Создание сайта www.abykov.dev](/assets/img/www-dns-records.png){: .shadow .rounded }

> ⚠️ Основной домен `abykov.dev` обслуживается GitHub Pages, а `www.abykov.dev` перенаправляет на него через сервер reg.ru.

## Локальная настройка и работа с блогом

Для установки темы нам потребуется:

- **Ruby** (>= 2.5)
- **Bundler**
- **Node.js** и **npm** (для управления ассетами)
- **Jekyll**
- **Git**

### Устанавливаем Ruby

```bash
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev
```

### Настройка переменных окружения для Ruby Gems

Откроем файл ```.bashrc```:
```bash
nano ~/.bashrc
```

В конец файла добавляем:
```bash
# Ruby Gems without sudo
export GEM_HOME="$HOME/.gem"
export PATH="$HOME/.gem/bin:$PATH"
```

Сохраняем (Ctrl+O → Enter → Ctrl+X) и применяем изменения:
```bash
source ~/.bashrc
```

Проверим переменные окружения:
  ```bash
echo $GEM_HOME
# должно вывести /home/твой_юзер/.gem
  
echo $PATH | grep gem
```

### Установим Jekyll и Bundler

```bash
ruby -v
gem install jekyll bundler
jekyll -v
```

### Установим Node.js и npm

```bash
sudo apt install nodejs npm
```

### Клонируем тему Chirpy

```bash
git clone https://github.com/cotes2020/jekyll-theme-chirpy.git abykov.dev
cd abykov.dev
bundle install
npm install
```
> ⚠️ В данном случае клонируется сама тема, которая содержит не только шаблоны, но и структуру проекта. 
> Такой подход дает полный контроль над содержимым и стилями сайта. Однако обновления темы нужно будет выполнять вручную, 
> сливая изменения из апстрима.

### Запуск локального сервера

```bash
bundle exec jekyll s
```

Теперь блог доступен по адресу:
```
http://localhost:4000
```

```
Configuration file: /home/aleksey/RubyProjects/abykov.dev/_config.yml
            Source: /home/aleksey/RubyProjects/abykov.dev
       Destination: /home/aleksey/RubyProjects/abykov.dev/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.687 seconds.
 Auto-regeneration: enabled for '/home/aleksey/RubyProjects/abykov.dev'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.

```

### Как редактировать посты
Работать с темой Chirpy (и любым другим Jekyll-сайтом) можно в любой среде, где установлены Ruby, Node.js и Jekyll. Например, в VS Code или RubyMine.
![Создание сайта www.abykov.dev](/assets/img/www-site-rubymine.png){: .shadow .rounded }

Все посты находятся в папке `_posts/`. Чтобы создать новый пост, нужно создать файл с именем вида `2025-05-01-some-name.md`
и в начале файла добавить front matter:

```yaml
---
title: "Название поста"
date: 2025-05-01 12:00:00 +0300
categories: [Блог]
tags: [jekyll, github-pages]
---
```
Далеее текст поста в формате Markdown.

После сохранения контент обновится динамически:
```text
Regenerating: 1 file(s) changed at 2025-05-01 16:09:50
              _posts/2025-04-30-deploy-blog.md
              ...done in 0.192125684 seconds.
              
Regenerating: 1 file(s) changed at 2025-05-01 16:10:01
              _posts/2025-04-30-deploy-blog.md
              ...done in 0.182268673 seconds.
              
Regenerating: 1 file(s) changed at 2025-05-01 16:10:26
              _posts/2025-04-30-deploy-blog.md
              ...done in 0.191651362 seconds.
```

## Настройки на стороне GitHub

После настройки локального окружения и домена осталось подготовить репозиторий на GitHub и опубликовать сайт.

### Создаём репозиторий

На GitHub создаём новый репозиторий, например abykov.dev. Если репозиторий уже существует - просто пропускаем этот шаг.

### Загружаем проект в репозиторий

В терминале, в каталоге проекта (abykov.dev), выполняем:
```bash
git init
git remote add origin git@github.com:username/abykov.dev.git
git add .
git commit -m "Initial commit: add Chirpy theme"
git push -u origin main
```
### Настраиваем GitHub Pages

GitHub Pages — это бесплатный хостинг от GitHub для статичных сайтов. Он позволяет выкладывать HTML, CSS и JavaScript, 
которые будут работать как полноценный сайт. Часто используется в связке с генераторами сайтов вроде Jekyll, Hugo, Astro 
и других.

Рендеринг в GitHub Pages происходит один раз при сборке (Build-time рендеринг), а не каждый раз при открытии сайта. 
Посты будут храниться в шаблонах Liquid для Jekyll, в Markdown-разметке, которую Jekyll-генератор будет транслировать 
в HTML/CSS/JS. Готовые страницы публикуются по адресу вида:
```
https://username.github.io
```

В репозитории на GitHub открываем Settings → Pages. Далее в разделе Build and deployment
в Source выбираем ветку ```gh-pages``` или ```main```.

В поле Custom domain вводим название основного домена -```abykov.dev```. Ставим галочку Enforce HTTPS,
чтобы GitHub автоматически выпустил и подключил SSL-сертификат для основного домена.

![Создание сайта www.abykov.dev](/assets/img/www-github-pages.png){: .shadow .rounded }

Теперь сайт будет доступен по адресу https://abykov.dev

### GitHub Actions и автоматический билд

Тема Chirpy использует GitHub Actions для автоматической сборки и публикации сайта.
Каждый раз, когда происходит пуш изменений в репозиторий, GitHub Actions запускает workflow, который
собирает сайт Jekyll и публикует результат на GitHub Pages.

Это упрощает работу и позволяет не выполнять сборку вручную.
> ⚠️ Обычно в репозитории Chirpy уже настроен файл ```.github/workflows/pages-deploy.yml```, который отвечает 
> за сборку и деплой.

### Проверка статуса сборки

После каждого коммита GitHub запускает GitHub Actions. В списке коммитов отображается статус:

- success - сборка и деплой прошли успешно.
- failure - ошибка сборки или деплоя. Нужно открыть вкладку Actions и посмотреть логи.
- in progress - сборка выполняется.

![Создание сайта www.abykov.dev](/assets/img/www-gp-success-status.png){: .shadow .rounded }

Чтобы увидеть детали, нужно перейти на вкладку Actions, кликнуть на последнем workflow run, посмотреть
логи build и deploy.
> ⚠️ Иногда может появиться статус Cancelled - это не всегда ошибка. Например, если новая сборка была запущена 
> до завершения предыдущей.
