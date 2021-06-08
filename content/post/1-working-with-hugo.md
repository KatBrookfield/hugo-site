---
title: "1 Working With Hugo"
date: 2021-06-07T22:33:20+01:00
archives: "2021"
tags: []
author: Kat Brookfield
---

I am using Hugo to generate static sites for this blog. As I am on a Mac using Brew, I will be using it to install Hugo. If you are using a different OS you can follow installation instructions on https://gohugo.io/getting-started/installing/

## Installation
Run following commands in your favourite terminal:

```
brew install hugo

hugo version
```

You should see output similar to this:
```
hugo version
hugo v0.83.1+extended darwin/arm64 BuildDate=unknown
```

## Selecting a Theme
There are a lot of Themes to choose from. This blog will be using a theme called [Jeffprod](https://themes.gohugo.io/hugo-blog-jeffprod/)

## Creating a Hugo site and applying theme
Execute following commands to create a new site and apply the selected theme:
```
hugo new site hugo-site

cd hugo-site

git clone https://github.com/Tazeg/hugo-blog-jeffprod.git themes/jeffprod
```
## Modify config.toml
In your working directory (hugo-site), modify *config.toml* file and change baseURL, title and add theme:
```
baseURL = "http://advecti.io"
languageCode = "en-us"
title = "Hi, I'm Hugo running on Kubernetes in a single pod!"
theme = "jeffprod"

[permalinks]
    post = "/:year/:filename"

[taxonomies]
    tag = "tags"
    archive = "archives"
```

## Create your post
Run following command to create a new post:
```
hugo new post/my-first-post.md
```

You can either edit the post in your terminal or use an editor, I am using Visual Studio Code at the moment as it let's you do a preview of your md file.
Once you're done editing your post, do not forget to change the value of *draft* to *false*, otherwise your post won't be published.

## Render your website
To see how your website will look run the following command:
```
hugo server
```

## Browse to your preview
In your browser, go to http://localhost:1313/ to see your website.

## Further changes to the config files
Depending on your Theme, you may want to adapt various configurations.
In my case, I have changed the following files so far:

1. In */themes/jeffprod/archetypes/post.md* changed Author
2. In */themes/jeffprod/layouts/partials/footer.html* changed Twitter link to include @MrsBrookfield and removed other socials

Now that we have the local site created we can proceed with putting it in a container.