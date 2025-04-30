---
title: "Развертывание блога на GitHub Pages с темой Chirpy"
layout: post
date: 2025-04-30 12:00:00 +0300
categories: [GitHub, Блог]
tags: [jekyll, github-pages, chirpy, ruby, deploy]
---

В этой статье рассмотрим, как развернуть персональный блог, например `abykov.dev`, на GitHub Pages с использованием готовой темы. В качестве основы возьмем тему **Chirpy** для Jekyll, которая, на мой взгляд, отлично подходит для минималистичного блога.

**Chirpy** — мощная и современная тема Jekyll с минималистичным дизайном, встроенным поиском, поддержкой категорий, тегов, комментариев, RSS и даже MathJax для формул.

Пошагово разберем, как клонировать тему, адаптировать ее под себя и опубликовать сайт — без лишней сложности и мистики.

## Привязка домена и редирект `www`

Для начала зарегистрируем домен у хостинг-провайдера, например на [reg.ru](https://www.reg.ru/), которому делегируем управление DNS-записями. Через этот же провайдер мы настроим SSL-сертификат и обеспечим редирект с `www.abykov.dev` на `abykov.dev`. При этом сам сайт будет хоститься на GitHub Pages — бесплатно и без серверов.

> ⚠️ Почему это важно?
GitHub Pages может обслуживать только **один кастомный домен** — либо `abykov.dev`, либо `www.abykov.dev`. Одновременная поддержка обоих с валидным HTTPS-сертификатом невозможна.

Поэтому мы «перехватываем» `www.abykov.dev` через своего хостинг-провайдера и на его стороне:

- выпускаем Let's Encrypt сертификат для `www.abykov.dev`;
- настраиваем веб-сервер так, чтобы он делал **301-редирект** на `https://abykov.dev`.

Это позволяет использовать короткий и каноничный адрес без `www`, при этом сохранить работоспособность и узнаваемость адреса с `www`.

## Что такое GitHub Pages

GitHub Pages — это бесплатный хостинг от GitHub для статичных сайтов. Он позволяет выкладывать HTML, CSS и JavaScript, которые будут работать как полноценный сайт. Часто используется в связке с генераторами сайтов вроде Jekyll, Hugo, Astro и других.

Рендеринг в GitHub Pages происходит один раз при сборке (Build-time рендеринг), а не каждый раз при открытии сайта. Посты будут храниться в шаблонах Liquid для Jekyll, в Markdown-разметке, которую Jekyll-генератор будет транслировать в HTML/CSS/JS. Готовые страницы публикуются по адресу вида:

```
https://username.github.io
```

## Установка окружения

Для установки темы нам потребуется:

- Ruby (>= 2.5)
- Bundler
- Node.js и npm (для управления ассетами)
- Jekyll
- Git (для клонирования)

### Устанавливаем Ruby

```bash
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev
```

### Добавим Ruby Gems в переменные окружения

```bash
nano ~/.bashrc
```

В конец файла добавляем:

```bash
# Ruby Gems without sudo
export GEM_HOME="$HOME/.gem"
export PATH="$HOME/.gem/bin:$PATH"
```

Сохраняем (Ctrl+O → Enter → Ctrl+X) и применяем:

```bash
source ~/.bashrc
```

Проверим переменные окружения:

```bash
echo $GEM_HOME
# должно вывести /home/твой_юзер/.gem

echo $PATH | grep gem
```

### Проверим Ruby и установим Jekyll

```bash
ruby -v
gem install jekyll bundler
jekyll -v
```

### Установим Node.js и npm

```bash
sudo apt install nodejs npm
```

## Клонируем тему Chirpy и запускаем локально

```bash
git clone https://github.com/cotes2020/jekyll-theme-chirpy.git abykov.dev
cd abykov.dev
bundle install
npm install
```

Запускаем локальный сервер:

```bash
bundle exec jekyll s
```

Теперь блог доступен по адресу:

```
http://localhost:4000
```

## Редактирование и среда разработки

Работать с темой Chirpy (и любым другим Jekyll-сайтом) можно в любой среде, где установлены Ruby, Node.js и Jekyll. Например, в VS Code или RubyMine.

---

Дальше мы настроим деплой, GitHub Actions и подключим кастомный домен.

