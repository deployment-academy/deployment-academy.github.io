# Deployment.properties

Source code for [deployment.properties](https://deployment.properties).

## Local Development

1. Clone this repository

    ```shell
    git clone git@github.com:deployment-academy/deployment-academy.github.io.git
    cd deployment-academy.github.io
    git submodule update --init --force # update themes
    ```

1. [Install Hugo](https://gohugo.io/getting-started/installing/)
1. Add a new post

    ```shell
    hugo new posts/name-your-post.md
    ```

1. Edit your post using your favorite editor
1. Run it locally to preview your changes

    ```shell
    hugo server
    ```

## Publishing

The content is published to the live blog on merges to the main branch.
For more details check [.github/workflows/gh-pages.yml](.github/workflows/gh-pages.yml).

Content preview is made locally following the steps on [Local Development](#local-development)

## Create a Hugo site

If you are interested in how this site was created and looking to create a
Hugo site, please check the blog post [Create a static website with Hugo, GitHub Pages and Actions (in minutes)](https://deployment.properties/posts/hugo/hugo-gh-pages-n-actions/).
