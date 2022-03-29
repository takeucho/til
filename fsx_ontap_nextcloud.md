# Amazon FSx for NetApp ONTAP と Nextcloud で実現するファイルサーバ兼クラウドストレージ環境

この記事では Amazon FSx for NetApp ONTAP と Nextcloud を使って、SMBファイルサーバ兼クラウドストレージとしてシームレスに使える環境構築の連携手順と構築時の注意点を紹介します。
最終的な想定構成は以下となります（本記事ではオンプレミス環境からのアクセスに関する設定については記載していません）。

![alt](https://github.com/takeucho/til/blob/main/images/fsx-ontap-nextcloud.png)

本記事は個人が作成したものであり、AWS社やNextcloud社とは一切関係ありません。

#### 対象読者
FSx for ONTAP、Amazon RDS、Amazon Elastic Load Balancing、Amazon ElastiCache、Nextcloud の知識を有している人。

#### この記事で説明しないこと
各 AWS サービスの構築手順や Nextcloud 自体の構築手順の詳細は説明していません。各種ドキュメントをご参照ください。

## FSx for ONTAP の構築
下記ワークショップの AWS Cloud Formation テンプレートを使用することで、Active Directory 含めた FSx for ONTAP 環境を構築できます。
https://github.com/aws-samples/amazon-fsx-workshop/tree/master/netapp-ontap/JP

下記共有を１つずつ作成します。

・NFS共有

・SMB共有

## Nextcloud サーバの構築
下記 Nextcloud のドキュメントを参照し、Ubuntu 20.04 LTS のAmazon EC2 インスタンスにNextcloudをインストールします。
https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html

#### 注意点
Snap Package を使用してインストールした場合、NextCloud をインストールしている Ubuntu への cifs-utils のインストールがうまくいかなかったため、FSx for ONTAP などの SMB ファイルサーバをマウントさせることはできませんでした。

## Nextcloud の標準ファイルディレクトリとして FSx for ONTAP の NFS 共有を使用する
今回、NextCloud は 2 台の EC2 で冗長化するため、標準コンテンツを格納するファイルディレクトリ（ /var/www/nextcloud/data/ ）として FSx for ONTAP の NFS 共有を使用する設定を行います。

Nextcloudをインストールした Ubuntu に NFS マウントに必要なツールをインストールします。

`sudo apt install nfs-common`

元のファイルディレクトリの中身を退避させます。

`sudo mv /var/www/nextcloud/data/ /var/www/nextcloud/data_back`

`sudo mkdir /var/www/nextcloud/data/`

FSx for ONTAP を NFS マウントします（FSx for ONTAP の NFS IP アドレス = 198.19.255.130 の場合）。

`sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 198.19.255.130:/vol1 /var/www/nextcloud/data/`

自動マウントの設定を行います。

`sudo echo "198.19.255.130:/vol1 /var/www/nextcloud/data/ nfs4 defaults 0 0" >> /etc/fstab`

退避させていたファイルを元のディレクトリに戻します。

`sudo cp -piR /var/www/nextcloud/data_back/. /var/www/nextcloud/data`

## Nextcloud でSMB 共有を使用する
上記の　NFS　共有だけでも Nextcloud を FSx for ONTAP をバックエンドストレージとしたクラウドストレージとして利用することができます。しかし、今回は Windows クライアントから使用したいため、FSx for ONTAP を SMB マウントできるように設定します。

Nextcloudをインストールした Ubuntu に SMB マウントに必要なツールのインストールを行います。

`sudo apt install cifs-utils`

`sudo apt install smbclient`

`sudo reboot`

`sudo service apache2 start`

Nextcloud で Active Directory への参加と SMB マウントができるようにアプリケーションをインストールします。
Nextcloud の管理者権限で GUI からログインし、Apps から "LDAP user and group backend" と "External storage support" をインストールします。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_apps.png)

設定画面の "LDAP/AD integration" から Active Directory への参加設定を行います。
その後、 設定画面の Administration の "External storage" から外部ストレージの設定として FSx for ONTAP をマウントする設定を行います。
また、"Allow users to mount external storage" と "SMB/CIFS" にチェックを入れることで、各ユーザ自身が自身のホームフォルダをマウントする設定ができるようになります。

![alt](https://github.com/takeucho/til/blob/main/images/external_storage.png)

上記設定を行うことで、Nextcloud の Files 画面で FSx for ONTAP の SMB 共有が "FSxN" として表示され、アクセスできるようになります。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_files.png)

## Amazon RDS の構築
上記で構築した Nextcloud サーバのデータベースを Amazon RDS に置きかえるため、下記ドキュメントを参照して Amazon RDS を構築します。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateDBInstance.html

Nextcloudをインストールした Ubuntu に Maria DB クライアントをインストールします。

`sudo apt install -y mariadb-client`

## Amazon ElastiCache の構築
メモリキャッシュを利用して高速化するために、下記ドキュメントを参照して Amazon ElastiCache を Redis で構築します。
https://aws.amazon.com/jp/getting-started/hands-on/building-fast-session-caching-with-amazon-elasticache-for-redis/1/

Redis の設定を config.php に追加します。

`sudo vi /var/www/nextcloud/config/config.php`

~~~
'memcache.distributed' => '\OC\Memcache\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => [
     'host' => 'nextcloud-redis.z6xxxx.ng.0001.apne1.cache.amazonaws.com',
     'port' => 6379,
~~~

## Amazon Elastic Load Balancing の構築
Nextcloud サーバを冗長化するため、下記ドキュメントを参照して Amazon Elastic Load Balancing を構築します。
https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/create-application-load-balancer.html

Elastic Load Balancing の設定を config.php に追加します。

`sudo vi /var/www/nextcloud/config/config.php`

~~~
'trusted_proxies' =>
array (
0 => '10.11.0.0/16',
),
'overwriteprotocol' => 'https',
'overwritehost' => 'ec2-xx-xxx-x-xxx.ap-northeast-1.compute.amazonaws.com',
~~~


## Nextcloud の設定
