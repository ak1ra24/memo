## mkdocs
### 自分のmkdocsのリポジトリ

[ak1ra memo](https://ak1ra24.github.io/memo/)

### git pushすると、travis ciで自動的に静的サイト生成＆gh-pagesブランチにpush

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

### 編集するところ
* `docs/` にmarkdownのwikiの内容
* `mkdocs.yml`にpagesを追加する

### 拡張機能について
* [使い方](https://ak1ra24.github.io/memo/howtowrite/)を参照


### 参考
[mkdocsについて](https://qiita.com/mebiusbox2/items/a61d42878266af969e3c)

## pandoc + Markdown

### 環境構築
WSL Ubuntu 18.04 + VSCODE

```
sudo apt update
sudo apt install -y pandoc pandoc-citeproc
## Tex Packege
sudo apt install texlive-luatex texlive-xetex texlive-lang-japanese -y
sudo apt install texlive-lang-cjk xdvik-ja evince texlive-fonts-recommended texlive-fonts-extra -y
## Haskell
sudo apt install haskell-platform -y
sudo apt install cabal-install -y
sudo cabal update
sudo cabal install --global pandoc-crossref

## WSLのUbuntu18.04で行うと、フォントがないのか？エラーを吐いて、一向にうまくいかないため、Windowsのパス上で行う
cd /mnt/c
mkdir pandoc
cd pandoc
code . 
```

上記でVSCode上でmarkdownを編集して、pdfを生成できる

### ディレクトリ構成

```
$ tree       
.
├── config.yml
├── imgs
│   ├── sample1.png
│   ├── sample2.png
│   └── sample3.png
├── makepdf.sh
├── myref01.bib
├── myref02.bib
├── paper.md
├── paper.pdf
├── sample.md
└── sample.pdf
```

### pandocでmarkdown -> latexpdf生成のshell
```
#!/bin/bash
CMDNAME=`basename $0`
if [ $# -ne 1 ]; then
  echo "Usage: $CMDNAME filename" 1>&2
  exit 1
fi

bibfiles=$(ls -F | grep bib)
echo 'bib files: '$bibfiles

pandoccmdtemplate='pandoc --pdf-engine=lualatex --toc -F pandoc-crossref -N -M "crossrefYaml=config.yml"'
echo $pandoccmd
pandoccmd=''
for bib in ${bibfiles[@]}
do
  addbib=' --bibliography='${bib}
  pandoccmdtemplate+=${addbib}
done

pandoccmdtemplate+=' '
pandoccmdtemplate+=$1.md
pandoccmdtemplate+=' -o '
pandoccmdtemplate+=$1.pdf

echo $pandoccmdtemplate

echo $1.md to $1.pdf

${pandoccmdtemplate}

echo Done
```

* `--toc`
  * 自動的に目次を生成
* `-N`
  * 番号付き節ヘッダーをつける
* `--bibliography`
  * 参照するファイルを読み取ってくれるが、変換前のmarkdownにbibのファイル名を書いても反応しない
  * 今回は、`ls -F | grep bib`でファイル名を取得して、ローカルの全`.bib`を適用させている

### Todo
??? Failure
    config.yamlが読み込まれておらず、figのままになっている

### 参考
<https://qiita.com/Kumassy/items/5b6ae6b99df08fb434d9>
<https://qiita.com/SOhtsu/items/e921142372d0f4cce410>