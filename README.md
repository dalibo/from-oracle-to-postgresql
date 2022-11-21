# From Oracle to PostgreSQL

## Yet another guide

This present guide aims to be a collaborative work, with a starting point former
and actual PostgreSQL enthousiasts from Dalibo, a french company. It focuses on
multiple differences between Oracle Databases and PostgreSQL in a developer's
point of view.

## Licenses

* This guide is released under the PostgreSQL license. See [LICENSE](LICENSE).

* The generated website theme "hugo-book" is released by Alex Shpak under the MIT
license. See [themes/hugo-book/LICENSE](themes/hugo-book/LICENSE).

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