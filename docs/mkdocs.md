## 自分のmkdocsのリポジトリ

[ak1ra memo](https://ak1ra24.github.io/memo/)

## git pushすると、travis ciで自動的に静的サイト生成＆gh-pagesブランチにpush

以下の `.travis.yml`でmasterにpushされると自動的にgh-pagesブランチに置いてくれる
```
language: python

install:
  pip install -r requirements.txt

script:
  mkdocs build --verbose

deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GITHUB_TOKEN
  local_dir: site
  on:
    branch: master
```

## 編集するところ
* `docs/` にmarkdownのwikiの内容
* `mkdocs.yml`にpagesを追加する

## 拡張機能について
* [使い方](https://ak1ra24.github.io/memo/howtowrite/)を参照


## 参考
[mkdocsについて](https://qiita.com/mebiusbox2/items/a61d42878266af969e3c)