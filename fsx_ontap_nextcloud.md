# Amazon FSx for NetApp ONTAP と Nextcloud で実現するファイルサーバ兼クラウドストレージ環境

この記事では Amazon FSx for NetApp ONTAP と Nextcloud を使って、SMB ファイルサーバ兼クラウドストレージとしてシームレスに使える環境構築の連携手順と構築時の注意点を紹介します。
最終的な想定構成は以下となります（本記事ではオンプレミス環境からのアクセスに関する設定については記載していません）。

![alt](https://github.com/takeucho/til/blob/main/images/fsx-ontap-nextcloud.png)

- 本記事は個人が作成したものであり、AWS 社や Nextcloud 社とは一切関係ありません。
- 本記事を参考に環境構築を行い何らかの障害や損害が発生しても一切の責任は負いませんので、自己責任でご参照ください。

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

## Amazon RDS の構築
上記で構築した Nextcloud サーバのデータベースを Amazon RDS に置きかえるため、下記ドキュメントを参照して Amazon RDS を構築します。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateDBInstance.html

Nextcloudをインストールした Ubuntu に Maria DB クライアントをインストールし、RDS にアクセスできることを確認します。

`sudo apt install -y mariadb-client`

Nextcloud が DB として RDS を参照するように config.php を修正します。

`sudo vi /var/www/nextcloud/config/config.php`

~~~
  'dbtype' => 'mysql',
  'version' => '22.2.3.0',
  'overwrite.cli.url' => 'http://10.11.0.46/nextcloud',
  'dbname' => 'nextcloud',
  'dbhost' => 'nextcloud-database-01.cnixxxxxxxx.ap-northeast-1.rds.amazonaws.com',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'ncdbuser',
  'dbpassword' => 'password',
  'installed' => true,
~~~

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
上記の NFS 共有だけでも Nextcloud を FSx for ONTAP をバックエンドストレージとしたクラウドストレージとして利用することができます。しかし、今回は Windows クライアントから使用したいため、FSx for ONTAP を SMB マウントできるように設定します。

Nextcloudをインストールした Ubuntu に SMB マウントに必要なツールのインストールを行います。

`sudo apt install cifs-utils`

`sudo apt install smbclient`

`sudo reboot`

`sudo service apache2 start`

Active Directory に 参加するための LDAP アプリケーションが使えるように php-ldap をインストールします。

`sudo apt install php-ldap`

`sudo service apache2 restart`

Nextcloud の管理者権限で GUI からログインし、Apps から "LDAP user and group backend" と "External storage support" をインストールします。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_apps.png)

設定画面の "LDAP/AD integration" から Active Directory への参加設定を行います。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_ldap.png)

設定画面の Administration の "External storage" から外部ストレージの設定として FSx for ONTAP をマウントする設定を行います。

また、"Allow users to mount external storage" と "SMB/CIFS" にチェックを入れることで、各ユーザ自身が自身のホームフォルダをマウントする設定が行えるようになります。

![alt](https://github.com/takeucho/til/blob/main/images/external_storage.png)

上記設定を行うことで、Nextcloud の Files 画面で FSx for ONTAP の SMB 共有が "FSxN" として表示され、アクセスできるようになります。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_files.png)

これにより、Nextcloud から FSxN フォルダに書き込んだファイルは Windows からマウントしている同一の SMB 共有から読み取り・変更ができます。また、同様に Windows のエクスプローラから SMB 共有に対して書き込んだファイルは、Nextcloud の FSxN フォルダで読み取り・変更ができます。 

WindowsのエクスプローラでマウントしたSMB共有を表示

![alt](https://github.com/takeucho/til/blob/main/images/Windows_share.png)

Nextcloud上のフォルダを表示

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_folder.png)

SMB共有とNextcloud上のフォルダの紐付け例

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_share.png)

## 多要素認証（MFA）の有効化
Nextcloud の "Two-Factor TOTP Provider" アプリケーションをインストールすることにより、Nextcloud のユーザを多要素認証で保護することができます。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_apps2.png)

インストール後の設定は、Personal の "Security" から "Enable TOTP" にチェックを入れることで QR コードが表示され、Google Authenticator などのスマートフォンの認証アプリと連携させることが可能です。連携させると、ログイン時に認証アプリで表示されるワンタイムパスワードの入力が必要となります。

![alt](https://github.com/takeucho/til/blob/main/images/nextcloud_totp.png)

## Nextcloud サーバの冗長化
Nextcloud サーバを冗長化するため、ここまで構成した EC2 を AMI として保存し、AMI から複製となる EC2 を作成します。

## Amazon Elastic Load Balancing の構築
上記 2 台の Nextcloud サーバを冗長化するため、下記ドキュメントを参照して Amazon Elastic Load Balancing を構築します。
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

以上
