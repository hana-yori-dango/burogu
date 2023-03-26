# Burogu

[![Rust](https://img.shields.io/badge/Rust-Rust?style=for-the-badge&logo=Rust&color=orange)](https://www.rust-lang.org/)
[![Zola](https://img.shields.io/badge/Zola-Zola?style=for-the-badge&color=black)](https://www.getzola.org/)

Made thanks to [https://www.getzola.org/themes/even/](https://www.getzola.org/themes/even/).

I have made minor updates such as:

- Github & LinkedIn icons in the navigation menu
- Added a description above post links
- Added tags to each post links
- Updated the post layout to have the tags at the beginning

## Getting Started

### Pre-requisites

- [Rust](https://www.rust-lang.org/tools/install) `1.68.1`
- [Zola](https://www.getzola.org/documentation/getting-started/installation/) `0.17.2`

### Development

Serves and re-renders the website as it is updated. [http://127.0.0.1:1111](http://127.0.0.1:1111)

```shell
zola serve
```

### Release

Builds the static website in `public/` directory.

```shell
zola build
```

## Deployment

[Cloudflare/pages](https://www.cloudflare.com/products/pages/) automatically deploies whenever I push on `main` branch.

`Cloudflare` failed to build with `zola 0.17.2` when I attempted it, so instead I build locally.

- Advantage: faster deploy since I build faster than `Cloudflare` CD.
- Drawback: have to do this manual step (or should have a pre-commit)
