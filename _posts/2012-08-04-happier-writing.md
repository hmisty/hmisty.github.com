---
layout: post
title: Happier writing
date: '2012-08-04'
author: Evan
tags:
- random
- happy
- writing
---

I am amazed by the good job by github pages with jekyll.
So happy writing is upgraded to happier writing! :-)

## tools
- markdown
- vim
- github pages

## howto
- vi to write in markdown
- git push to github
- done!

## howto better
However, when I turned to design my style of the site, I do not want to git push to github after each trival changes. It is better to install jekyll for local testing.

I am the first time trying Ruby powered tool and I am amazed about how easy it is. Good job!

Here describes my painless journey:

1. USE ubuntu 12.04 (important for being painless! do not try windows :)
2. sudo apt-get install rubygems
3. sudo gem install jekyll rdiscount <br>
   if you are behind a proxy, try: sudo gem install --http-proxy http://proxy-ip:proxy-port jekyll rdiscount
4. git clone git@github.com:hmisty/hmisty.github.com.git <br>
   cd hmisty.github.com
5. (terminal 1) jekyll --server <br>
   (terminal 2) vim <br>
   (terminal 3) run jekyll everytime you modified the document to re-generate the _site & refresh the browser browsing http://localhost:4000

Enjoy!
