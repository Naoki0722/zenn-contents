# My Zenn articles

https://zenn.dev/naoki0722

# Zenn CLI

- [ð How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## New Article

```
npx zenn new:article --slug [permalink] --title [title] --type [idea or tech] --emoji â¨
```

â»slug ã¯ a-z0-9ããã¤ãã³-ãã¢ã³ãã¼ã¹ã³ã¢\_ã® 12ã50 å­ã®çµã¿åããã«ããå¿è¦ãããã¾ãã

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

