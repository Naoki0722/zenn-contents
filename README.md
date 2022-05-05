# My Zenn articles

https://zenn.dev/naoki0722

# Zenn CLI

- [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## New Article

```
npx zenn new:article --slug [permalink] --title [title] --type [idea or tech] --emoji ✨
```

※slug は a-z0-9、ハイフン-、アンダースコア\_の 12〜50 字の組み合わせにする必要があります

## New Book

```
npx zenn new:book --slug [slug name]
```

## Zenn Preview

```
npx zenn preview
```

# Link Check for markdown

```
markdown-link-check ./articles/** --config "markdown-link-check.json"
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

```
npm run lint
```


## Appendix

VSCode Markdown plugin

https://zenn.dev/ctrlkeykoyubi/articles/vscode-markdown-all-in-one

