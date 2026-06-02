# Ludovic Henry's Blog

This repository contains the source for a Jekyll site hosted with GitHub Pages.

## Run Locally With Docker

These steps run the site inside a fresh `ubuntu:24.04` Docker image.

### Prerequisites

- Docker

### Start an Ubuntu 24.04 Container

From the repository root, start an interactive container with the current directory mounted at `/site`:

```sh
docker run --rm -it -p 4000:4000 -v "$PWD:/site" -w /site -e PAGES_REPO_NWO=luhenry/luhenry.github.io ubuntu:24.04 bash
```

### Install Dependencies

Inside the container, install Ruby, build tools, and Bundler:

```sh
apt-get update
apt-get install -y ruby-full build-essential git zlib1g-dev
gem install bundler
bundle install
```

### Start the Development Server

Inside the container, run Jekyll and bind it to all container interfaces:

```sh
bundle exec jekyll serve --host 0.0.0.0
```

Open the site from the host machine at `http://127.0.0.1:4000`.

### Include Draft Posts

To preview posts in `_drafts`, start the server with drafts enabled:

```sh
bundle exec jekyll serve --drafts --host 0.0.0.0
```

### Build the Site

Generate the static site into `_site`:

```sh
bundle exec jekyll build
```

