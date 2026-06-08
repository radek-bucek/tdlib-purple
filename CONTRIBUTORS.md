# Contributors

## Packaging

Packaged and maintained by Radek Bucek ([@radek-bucek](https://github.com/radek-bucek)).

This repository contains only the Fedora packaging of an existing plugin:
the `tdlib-purple.spec` RPM spec, the `make-srpm.sh` build wrapper, the two
patches, the helper scripts and the documentation — together with their
maintenance (Telegram API registration, RPM signing, testing on Fedora 43
KDE). The plugin itself and TDLib are upstream work, licensed under GPL-2.0
and credited below.

## Upstream code

This repository packages and patches the following upstream projects; their
authors are credited in their own commit histories and license files:

- [`tdlib-purple`](https://github.com/ars3niy/tdlib-purple) — original
  libpurple plugin by Arseniy Lartsev
- [`BenWiederhake/tdlib-purple`](https://github.com/BenWiederhake/tdlib-purple)
  — actively maintained fork (collects working PRs against the original),
  the actual code base shipped in this RPM
- [`tdlib/td`](https://github.com/tdlib/td) — Telegram Database Library by
  the Telegram team, statically linked at version 1.8.35
