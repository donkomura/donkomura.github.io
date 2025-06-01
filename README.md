# donko's blog

donkomura の技術ブログ

## How to publish

- `content` に記事を markdown で置く
- Pull Request を作成して `main` ブランチにマージする
    - GitHub Actions で GitHub Pages にデプロイされる

## Development 

### Prerequisites

- [Zola](https://www.getzola.org/)

### How to run on local

```bash
zola serve
```

表示されるアドレスにアクセスするとページが表示されます。
記事を保存すると自動でページに反映されます。
