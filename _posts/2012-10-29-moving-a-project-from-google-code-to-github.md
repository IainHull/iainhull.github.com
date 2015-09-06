---
layout: post
title: "Moving a project from google code to github"
description: ""
subtitle: "Finally moved my old cmakeant project from google-code to github (I have been meaning to do this for ages). In case you are trying to do the same, here are the steps."
category: github 
tags: [git, hg, github, google-code]
---

Finally moved my old cmakeant project from google-code to github (I have been meaning to do this for ages). In case you are trying to do the same, here are the steps:

1. Download the mercurial repo from google code

        hg clone https://iainhull@code.google.com/p/cmakeant/ cmakeant-hg

1. Use fast-export to convert the mercurial repository to git.  Note: I needed to add mercurial on my `PYTHONPATH`
   
        git clone git://repo.or.cz/fast-export.git
        export PYTHONPATH=/usr/local/Cellar/mercurial//2.3.1/libexec/:$PYTHONPATH    

1. Create a new git repo and clone the mercurial repo

        git init cmakeant-git
        cd cmakeant-git
        ../fast-export/hg-fast-export.sh -r ../cmakeant-hg/
    
1. Create .gitignore, readme and license and commit any changes

        cp .hgignore .gitignore
        vi .gitignore
        vi README.md
        wget http://www.apache.org/licenses/LICENSE-2.0.txt
        git add -n .
        # verify files added
        git add .
        git commit
    
1. Create a new repo on github (using the web interface).

1. Set the origin on your local repo and push the changes to github

        git remote add origin https://github.com/IainHull/cmakeant.git
        git push -u origin master
 
1. Finally move the issues from the google code tracker to github.  (I found some alpha code that could do this, but decided to import the few issues I had manually).
