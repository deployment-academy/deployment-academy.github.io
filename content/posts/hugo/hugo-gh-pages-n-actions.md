---
title: "Create a static website with Hugo, GitHub Pages and Actions (in minutes)"
description: "In this tutorial, we are going to get a static website up and running with Hugo, GitHub Pages and Actions."
date: 2021-08-15T22:55:44-04:00
lastmod: 2021-08-16
draft: false
tags:
  - hugo
---

In this tutorial, we are going to get a static website up and running with Hugo.
We will deploy the site using GitHub Pages and make sure new changes are
automatically published using GitHub Actions.

<!--more-->

Hugo is a static site generator written in Go that helps you to organize and
manage content without the need of server-side rendering, databases, and etc.
Hugo is specially interesting if you want to create, maintain and version your
content in GitHub using Markdown.

Check out the Hugo website to learn more about [what it is](https://gohugo.io/about/what-is-hugo/)
and [the benefits of using a static site generator](https://gohugo.io/about/benefits/).

The Hugo documentation is pretty good and has everything you need to get started
and publish your site to different providers. I'll describe here the steps I
followed to put this wesbite together in a few minutes.

## Install Hugo

There are different installation methods in [the documentation](https://gohugo.io/getting-started/installing/).
I prefer a simple download:

```bash
# check https://github.com/gohugoio/hugo/releases for the last release
hugo_version=0.87.0
wget https://github.com/gohugoio/hugo/releases/download/v"$hugo_version"/hugo_"$hugo_version"_macOS-64bit.tar.gz
tar -xzvf hugo_"$hugo_version"_macOS-64bit.tar.gz hugo
mv hugo /usr/local/bin
rm hugo_"$hugo_version"_macOS-64bit.tar.gz
```

Check if everything is ok:

```bash
hugo version
```

## Build the website

Create the site:

```bash
site_name=your-site
hugo new site "$site_name"
```

Choose a theme in the [galery of themes](https://themes.gohugo.io/) and add it
to your site:

```bash
theme_name=beautifulhugo
theme_git=https://github.com/halogenica/beautifulhugo.git

cd "$site_name"
git init
git submodule add "$theme_git" themes/"$theme_name"
```

Add the theme to your configuration:

```bash
echo theme = \""$theme_name"\" >> config.toml
```

The `config.toml` file is where you configure the many Hugo parameters. Check
the [configuration page](https://gohugo.io/getting-started/configuration/) for
more details and available settings.

Start the local server:

```bash
hugo server -D
```

Hugo will initialize a server and watch for changes. You can keep this session
running as you interact with your site in the browser, so open a new session for
the next commands. Check the site on the web browser: `http://localhost:1313`

Add a new post:

```bash
hugo new posts/build-a-hugo-site.md
```

Edit this file:

```bash
# use your preferred editor
vim content/posts/build-a-hugo-site.md
```

Add anything you want to this file but make sure you change the `draft` option
in the [front matter config](https://gohugo.io/content-management/front-matter/)
to `draft: false`.

Note that the browser has already been updated with the new content.
This hot reload is really useful to make the creation process more interactive
and efficient.
You can add more content if you want. In the next session, you will publish this
site to GitHub Pages.

## Publishing to GitHub Pages

I believe GitHub Pages is a feature that doesn't need introductions. If you want
to know more about it, please check https://pages.github.com.

The Hugo documentation should have all you need to know to [deploy Hugo to GitHub
Pages](https://gohugo.io/hosting-and-deployment/hosting-on-github/). Here are
the steps I followed.

First, create a repository using your GitHub username: `<your-username>.github.io`.

One option is to generate the static content locally with `hugo -D` and push the
content of the `public` folder to the root of this repository. That would be all
you need. But **don't do that**, follow the following steps to enable a
publishing Git workflow.

Push the existing code to GitHub:

(If you generated the static content make sure you remove the `public/` folder
proceeding).

```bash
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-username>.github.io.git
git push -u origin main
```

Create the [GitHub Actions](https://docs.github.com/en/actions) config in the
file `.github/workflows/gh-pages.yml`:

```yaml
name: github pages
on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:
jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true
      - name: Build
        run: hugo --minify
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

This config is using the [hugo-setup](https://github.com/marketplace/actions/hugo-setup)
and [github-pages](https://github.com/marketplace/actions/github-pages-action)
actions. Check the docs for more options and details.

Push this change:

```bash
git add .
git commit -m "add workflow for hugo publishing and gh-pages"
git push -u origin main
````

Go to your Pages configuration (`https://github.com/<your-username>/<your-username>.github.io/settings/pages`)
and make sure you configure the GitHub Pages to serve content from the
`gh-pages` branch. You should be able to access your website using the URL
`<your-username>.github.io`.

Now, every time you push a new post or change to the `main` branch, the GitHub
Action will build your website and publish it.

## (Optional) Add a custom domain

Create a file in `static/CNAME` with the name of your domain.

```bash
echo -n your-domain.com > static/CNAME
```

Push this change as well and check the [GitHub Pages documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
for more details about how to configure your domain. There are a few options
available.

If you are using an apex domain (a root level domain), you can add the following
DNS records to your DNS manager:

|Record|Host Record|Points To|TTL|
|--|-----------|---------|---|
|A|@|185.199.108.153|4 Hours|
|A|@|185.199.109.153|4 Hours|
|A|@|185.199.110.153|4 Hours|
|A|@|185.199.111.153|4 Hours|
|CNAME|www|your-domain.com|4 Hours|

In addition to hosting your pages, GitHub will also provision a TLS certificate
for your domain.
