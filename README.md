# gmelodie.com
Gabe's portfolio/blog website based on the [hugo-coder](https://github.com/luizdepra/hugo-coder/) hugo theme.

Obs: when cloning also clone the hugo template (it's a git submodule) with
```
git submodule update --init --recursive
```

Run it locally with `hugo serve`

Obs: a post needs `public = "true"` in its front matter to show up on the `/posts/` list. Any other value hides it from the list, but the post URL, the RSS feed and the sitemap still serve it.
