# From Oracle to PostgreSQL

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]
[![GitHub Workflow Status][gh-pages-badge]][gh-pages]
[![GitHub issues][gh-issues-badge]][gh-issues]
[![GitHub pull requests][gh-pr-badge]][gh-pr]

[gh-pages]: https://dalibo.github.io/from-oracle-to-postgresql/
[gh-pages-badge]: https://img.shields.io/github/actions/workflow/status/dalibo/from-oracle-to-postgresql/gh-pages.yml?label=gh-pages

[gh-issues]: https://github.com/dalibo/from-oracle-to-postgresql/issues
[gh-issues-badge]: https://img.shields.io/github/issues/dalibo/from-oracle-to-postgresql

[gh-pr]: https://github.com/dalibo/from-oracle-to-postgresql/pulls
[gh-pr-badge]: https://img.shields.io/github/issues-pr/dalibo/from-oracle-to-postgresql

## Yet another guide

This present guide aims to be a collaborative work, with a starting point former
and actual PostgreSQL enthousiasts from Dalibo, a french company. It focuses on
multiple differences between Oracle Databases and PostgreSQL in a developer's
point of view.

## Licenses

* This guide is released under the CC BY-NC-SA license. See [LICENSE](LICENSE).

* The generated website theme "hugo-book" is released by Alex Shpak under the MIT
license. See [themes/hugo-book/LICENSE](themes/hugo-book/LICENSE).

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg

## How to contribute

Any contribution on the project is welcome. Create an issue for misspelling word
or erroneous information or open a new Pull Request to add or modify any content
of this guide.

Content is written in pure markdown with 80 characters per line.

This guide provides several translations under appropriate `content.xx`
directory, where `xx` is a language subtag according to [RFC
5646](https://www.rfc-editor.org/rfc/rfc5646). A new language must be added in
global [config.yaml](config.yaml), as follows:

```yaml
languages:
  en:
    languageName: English
    contentDir: content.en
    weight: 1
    title: From Oracle To PostgreSQL
  fr:
    languageName: French
    contentDir: content.fr
    title: Porter Oracle vers PostgreSQL
    weight: 2
```