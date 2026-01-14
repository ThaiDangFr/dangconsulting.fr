update site
===========
- Push to master => Automatically redeploys and updates the gh-pages branch.
- I have protected the master branch and created a develop branch.

Merging updates from the original version
```bash
git remote add template git@github.com:alshedivat/al-folio.git
git fetch template
git checkout develop
git merge template/master --allow-unrelated-histories
# => Open Sublime Merge to choose 'theirs' or 'ours'
```
