# [ARM Technical Documentation](https://georglauterbach.github.io/arm/)

## :page_with_curl: About

This repository contains a personal documentation about ARM. The information is mostly taken from the [ARM Developer Documentation], but other sources are used as well. The documentation is built with [MkDocs Material] and deployed on [GitHub Pages].

[ARM Developer Documentation]: https://developer.arm.com/documentation/
[MkDocs Material]: https://squidfunk.github.io/mkdocs-material/
[GitHub Pages]: https://pages.github.com/

## :package: Serving Locally

To build and view the documentation locally, run the following command to serve the documentation on port 8000 (and make it reachable for other hosts in the network too).

```console
$ docker run --rm -it -p 0.0.0.0:8080:8000 -v "${PWD}/documentation:/docs" docker.io/squidfunk/mkdocs-material:9.1.5
INFO     -  Building documentation...
INFO     -  Cleaning site directory
INFO     -  Documentation built in ... seconds
INFO     -  [...] Watching paths for changes: 'content', 'mkdocs.yml'
INFO     -  [...] Serving on http://0.0.0.0:8000/
```
