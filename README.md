documentation
=============
- https://shopify.github.io/liquid
- https://jekyllrb.com/docs/step-by-step/01-setup => intéressant pour comprendre
- https://github.com/mnyrop/nycdh-jekyll/blob/master/docs/markdown-cheatsheet.md => tables, images
- emoji
  - https://www.webfx.com/tools/emoji-cheat-sheet
  - https://emojidb.org/stock-market-emojis
- mathjax https://jojozhuang.github.io/tutorial/mathjax-cheat-sheet-for-mathematical-notation/


dashboard
=========
- google analytics : https://analytics.google.com/
- search console   : https://search.google.com/       => scrap sitemap


al-folio
========
- https://github.com/alshedivat/al-folio
- exemple : https://alshedivat.github.io/al-folio/

git
===
- push vers master => redeploie automatiquement et met à jour la branche gh-pages
- j'ai protégé la branche master, et créé une branche develop

merger avec les nouveautés de la version originale
```bash
git remote add template git@github.com:alshedivat/al-folio.git
git fetch template
git checkout develop
git merge template/master --allow-unrelated-histories
=> ouvrir sublime merge pour choisir theirs ou ours
```

dev en local
============
http://localhost:8080

```
$ docker compose pull
$ docker compose up
$ docker compose up --build
```

- pour exclure des fichiers du rebuild automatique, voir dans \_config.yml dans la section "exclude:"

analyse de l'existant
=====================
- about             => _pages/about.md et utilise le _layouts/about.liquid
- blog              => _pages/blog.md et _posts/*.md
- publications      => supprimer publications.md
- projects          => transformer en "Stocks"
- repositories      => supprimer repositories.md
- cv                => supprimer cv.md
- teaching          => supprimer teaching.md
- people            => supprimer profiles.md et about_einstein.md
- submenus          => supprimer dropdown.md

cible
=====
- stocks
  - tableau avec "ticker" et "nom"
    - "ticker" renvoie sur une page avec "description" et "sector"
- blog

giscus
======
- sur github, activer settings > discussions
- https://github.com/marketplace/giscus et cliquer install
- aller sur https://giscus.app/fr pour récuppérer repo_id et category_id
- personnaliser la section giscus dans \_config.yml et changer category à Announcements

stock list
==========
- https://cloudcannon.com/cheat-sheets/jekyll/
- https://jekyllrb.com/docs/collections/

On met les actions dans \_projects pour bénéficier du moteur de recherche
J'ai essayé de le mettre dans un autre répertoire, le moteur de recherche ne l'a pas indexé


Pipeline
========
Prettier code formatter / check (pull_request) :
- mettre des exclude dans .prettierignore
- il veut une ligne vide à la fin des fichiers

```bash
npm install -D prettier @shopify/prettier-plugin-liquid
npx prettier . --check
npx prettier . --write
```
