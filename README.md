# [Rico's blog](https://blog.rico-j.de) - IT, infosec, osint, malware research, etc.

Hi there and welcome to the GitHub repository of [my blog](https://blog.rico-j.de). I am using GitHub pages with the [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) Jekyll theme to host said blog.
I am writing about a lot of things I come across. Personal stuff, opinions, interesting tech, malware, infosec, osint, and a lot more.

## Structure of the repository

The structure of this repository is actually quite simple.
Check out this directory tree.

```
.
├───assets
│   ├───files
│   │   ├───<dirs for other files used in articles>
│   │   ├───yyyy-mm-dd-name-of-article
│   │   └───yyyy-mm-dd-name-of-article-n
│   └───img
│       ├───<dirs for image files used in articles>
│       ├───yyyy-mm-dd-name-of-article
│       └───yyyy-mm-dd-name-of-article-n
├───_includes
│   └───head
├───_pages
└───_posts
    ├───<all article markdown files>
    ├───yyyy-mm-dd-name-of-article-1.md
    └───yyyy-mm-dd-name-of-article-n.md
```

All the articles go into the `/_posts` directory.
They are named after this schema: `yyyy-mm-dd-name.md`.

For each article there can be matching directories with the same name (excluding the `.md`) as the article within the `/assets/files` and `/assets/img` directories.
In those go all the data/images I refer to in my posts.

