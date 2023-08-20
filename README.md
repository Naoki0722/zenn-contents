# My Zenn articles

https://zenn.dev/naoki0722

# Zenn CLI

- [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## New Article

```bash
$ npx zenn new:article --slug [permalink] --title [title] --type [idea or tech] --emoji ✨ --published true --publication-name [組織名]
```

※slug は a-z0-9、ハイフン-、アンダースコア\_の 12〜50 字の組み合わせにする必要があります。

## New Book

```bash
$ npx zenn new:book --slug [slug name]
```

## Zenn Preview

```bash
$ npx zenn preview
```

# Link Check for markdown

```bash
$ npm run markdown-check
#(markdown-link-check ./articles/** --config "markdown-link-check.json")
```

## setting link check ignore patterns

Edit markdown-link-check.json

```
  "ignorePatterns": [
    {
      "pattern": "^http://localhost",
      "comment": "Ignore local tensorboard links"
    }
  ]
```

# Textlint

```bash
$ npm run lint
```


## Appendix

VSCode Markdown plugin

https://zenn.dev/ctrlkeykoyubi/articles/vscode-markdown-all-in-one
