# Mawile Detection

## Training Setup

スポットインスタンスのリクエスト

インスタンスの作成
* ステップ1 
    * AMI：AmazonLinux2の
* ステップ2
    * GPU コンピューティング: p2.xlarge
* ステップ3: インスタンスの詳細の設定
    * スポットインスタンスのリクエストにチェックを入れる
    * 最大価格の設定
        * 現在の価格という欄が現れて、ap-northeast-1a:$0.4626などが表敬式で表示されている
        * この表示価格より高い金額を設定する
            * 上記の場合、0.5だとエラーとなり、0.6でうまく作成できたので、ギリギリを攻めすぎるとダメかも
* セキュリティグループの8000を許可
* 作成されたインスタンスのグローバルアドレスにSSHして、下記のコマンドを実行する

```
[ec2-user@ip-x-x-x-X ~]$ sudo yum -y upgrade
[ec2-user@ip-x-x-x-X ~]$ sudo yum -y install git
[ec2-user@ip-x-x-x-X ~]$ sudo yum -y install git tmux emacs gcc gcc-c++ python-setuptools python-devel 
[ec2-user@ip-x-x-x-X ~]$ sudo git clone https://github.com/yyuu/pyenv.git /usr/bin/.pyenv
[ec2-user@ip-x-x-x-X ~]$ cd /usr/bin/.pyenv
[ec2-user@ip-x-x-x-X ~]$ sudo mkdir shims
[ec2-user@ip-x-x-x-X ~]$ sudo mkdir versions
[ec2-user@ip-x-x-x-X ~]$ sudo chown -R ec2-user:ec2-user /usr/bin/.pyenv
[ec2-user@ip-x-x-x-X ~]$ vi ~/.bashrc

# 下記を末尾に追記
# from
export PYENV_ROOT="/usr/bin/.pyenv"
if [ -d "${PYENV_ROOT}" ]; then
export PATH=${PYENV_ROOT}/bin:$PATH
eval "$(pyenv init -)"
fi
# to

[ec2-user@ip-x-x-x-X ~]$ source ~/.bashrc
[ec2-user@ip-x-x-x-X ~]$ pyenv install --list
``` 

で、ズラッと一覧が表示されればOK
上記の一覧からanacondaの最新バージョンを選択。
筆者の場合は`3-5.2.0`

```
[ec2-user@ip-x-x-x-X ~]$ pyenv install anaconda3-5.2.0
[ec2-user@ip-x-x-x-X ~]$ pyenv global anaconda3-5.2.0
[ec2-user@ip-x-x-x-X ~]$ python --version
```

ここで、`Python 3.6.5 :: Anaconda, Inc.` と表示されていればOK

```
[ec2-user@ip-x-x-x-X ~]$ conda install keras
[ec2-user@ip-x-x-x-X ~]$ conda install tensorflow
[ec2-user@ip-x-x-x-X ~]$ conda install -c conda-forge jupyterhub
[ec2-user@ip-x-x-x-X ~]$ cd /home/ec2-user/
[ec2-user@ip-x-x-x-X ~]$ mkdir jupyterhub
[ec2-user@ip-x-x-x-X ~]$ cd jupyterhub/
[ec2-user@ip-x-x-x-X ~]$ jupyterhub --generate-config
Writing default config to: jupyterhub_config.py
[ec2-user@ip-x-x-x-X ~]$  vi jupyterhub_config.py

#c.Spawner.notebook_dir = ''
c.Spawner.notebook_dir = '~/notebook'

[ec2-user@ip-x-x-x-X ~]$ sudo passwd ec2-user
[ec2-user@ip-x-x-x-X ~]$ sudo reboot
```
再起動まで待ち、再びSSHする。

```
[ec2-user@ip-x-x-x-X ~]$ jupyterhub -f /etc/jupyterhub/jupyterhub_config.py　&

```

ip address:8000にアクセスするとjupyterhubのログイン画面が表示されるので、ec2-userでログイン（パスワードは上記で設定したもの）できたらOK

```
[ec2-user@ip-x-x-x-X ~]$ git clone https://github.com/sacchin/mawile_ssd_keras.git
[ec2-user@ip-x-x-x-X ~]$ cd mawile_ssd_keras/
[ec2-user@ip-x-x-x-X ~]$ git checkout add_training_code
```
こっからダウンロードしたzipファイルを送信

```
[ec2-user@ip-x-x-x-X ~]$ mv /home/ec2-user/images.zip /home/ec2-user/mawile_ssd_keras/data
[ec2-user@ip-x-x-x-X ~]$ cd /home/ec2-user/mawile_ssd_keras/data
[ec2-user@ip-x-x-x-X ~]$ rm images/ -r
[ec2-user@ip-x-x-x-X ~]$ unzip images.zip
```


* 再びjupyterhubにログインして、mawile_ssd_keras/kucheat_training.pyを開く
    * 211行目: batch_size = 10
    * 260行目: nb_epoch = 10
* training.ipynbを開いて、2行目までを実行する
    * /home/ec2-user/mawile_ssd_keras/data/weights
    * /home/ec2-user/mawile_ssd_keras/data/checkpoints

## Web Application Setup
* 静的ファイルの参照は、nginxの設定が一番の問題でした。
* 事前にDocker Hubでレポジトリを作成しておく

```
$ docker tag flaskapp_uwsgi sacchin/mawile_detector:latest
$ docker push sacchin/mawile_detector:latest
$ docker tag flaskapp_nginx sacchin/mawile_detector_nginx:latest
$ docker push sacchin/mawile_detector_nginx:latest
```

## 参考
* https://qiita.com/michimani/items/fc64dcbe721d91579ccb
* https://jp.ubunifu.co/python/jupyterhub-pyenv-anaconda%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89%EF%BC%88aws-ec2%EF%BC%89
* https://haitenaipants.hatenablog.com/entry/2018/06/21/210108
