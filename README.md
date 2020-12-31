# Prerequisites

## Running locally (Ruby + Jekyll)

If running the site locally, install Ruby and Jekyll:

https://jekyllrb.com/docs/installation/windows/

## Running on Docker

Alternatively, you can use [docker](https://docs.docker.com/docker-for-windows/install/) if you don't want to install Ruby and Jekyll.

On Windows, make sure to install [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package).

# Blog Development (local)

To build the site and make it available on a local server:

```
bundle exec jekyll server
```

# Blog Development (docker)

To launch the server, use docker-compose:

```
docker-compose up
```

The blog will then be available at localhost:4000.

To shutdown the server:

```
docker-compose down
```

