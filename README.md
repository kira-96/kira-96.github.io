<h1 align="center">kira's blog</h1>

![Hugo version](https://img.shields.io/badge/Hugo-latest-blue?logo=Hugo)
[![hugo-papermod](https://img.shields.io/badge/Hugo--Themes-@PaperMod-blue)](https://themes.gohugo.io/themes/hugo-papermod/)
![GitHub Workflow Status (with branch)](https://img.shields.io/github/actions/workflow/status/kira-96/kira-96.github.io/pages.yml?branch=main&logo=github&style=flat-square)
![GitHub last commit (branch)](https://img.shields.io/github/last-commit/kira-96/kira-96.github.io/main?style=flat-square)
[![GitHub license](https://img.shields.io/github/license/kira-96/kira-96.github.io?style=flat-square)](https://github.com/kira-96/kira-96.github.io/blob/src/LICENSE)

Use [Hugo](https://gohugo.io/) & theme [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/)

### Clone

``` bash
$ git clone --recurse-submodules https://github.com/kira-96/kira-96.github.io.git
```

### Build

``` bash
$ hugo -D server
```

### Update Theme

``` bash
$ git submodule update --remote themes/PaperMod
```

### New draft

``` bash
$ hugo new posts/my-first-post.md
```
