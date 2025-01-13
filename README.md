Site will build in `_site`. It should be a copy of the git repo on the `master` branch.

    jekyll build

then

    cd _site
    git commit
    git push

or copy-paste this:

    jekyll build && cd _site/ && git commit -av && git push
