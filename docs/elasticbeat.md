# make elastic beat

## Beatの構造

![Beat overview architecture](https://www.elastic.co/guide/en/beats/devguide/current/images/beat_overview.png)



## 環境

* go 1.13.11
* python 3.6.9



## 手順

1. 環境をつくる

   ```
   sudo apt remove 'golang-*'
   wget https://dl.google.com/go/go1.13.11.linux-amd64.tar.gz
   tar xf go1.13.11.linux-amd64.tar.gz
   sudo mv go /usr/local/go
   mkdir -p $HOME/go
   export GOPATH=$HOME/go >> ~/.bashrc
   export GOROOT=/usr/local/go >> ~/.bashrc
   export PATH=$GOPATH/bin:$GOROOT/bin:$PATH >> ~/.bashrc
   sudo apt install -y python3-dev python3-pip
   pip3 install virtualenv
   export PYTHON_EXE='python3'
   export VIRTUALENV_PARAMS='-p python3'
   export VIRTUALENV_PYTHON='/usr/bin/python3'
   ```

   

2. mageをインストール

   ```
   go get -u -d github.com/magefile/mage
   cd $GOPATH/src/github.com/magefile/mage
   go run bootstrap.go
   // mageのバイナリが、$HOME/go/bin配下にある
   ```

   

3. mageを使ってbeatのテンプレートを作成

   ```bash
   mkdir -p ${GOPATH}/src/github.com/{user}
   mkdir -p ${GOPATH}/src/github.com/elastic
   git clone https://github.com/elastic/beats ${GOPATH}/src/github.com/elastic/beats
   cd  $GOPATH/src/github.com/elastic/beats/
   mage GenerateCustomBeat
   ```

   ```
   Enter the beat name [examplebeat]: Countbeat
   Enter your github name [your-github-name]: ak1ra24
   Enter the beat path [github.com/ak1ra24/countbeat]: 
   Enter your full name [Firstname Lastname]: Taro Hoge
   Enter the beat type [beat]: 
   ```

   

4. makeでバイナリ生成

   ```
   cd ${GOPATH}/src/github.com/{user}
   make
   ```

5. コードを編集

   ```
   config/config.go // configファイルで読み込む構造体を設定
   beater/xxxbeat.go // 送信するイベントの値を設定および送信を書く
   ```

   

   refs:

   [Beat overview](https://www.elastic.co/guide/en/beats/devguide/current/newbeat-overview.html)

   [Beatを作るコード書き方](https://qiita.com/datake914/items/d936cf866f78d7aa9393)

   [Beatのテンプレート生成の環境構築](https://www.cnblogs.com/sanduzxcvbnm/p/12091152.html)

   
