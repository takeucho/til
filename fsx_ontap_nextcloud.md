# Amazon FSx for NetApp ONTAP と Nextcloud で実現するファイルサーバ兼クラウドストレージ環境

この記事では Amazon FSx for NetApp ONTAP と Nextcloud を使って、SMBファイルサーバ兼クラウドストレージとしてシームレスに使える環境構築の概要手順と構築時の注意点を紹介します。
最終的な想定構成は以下となります（本記事ではオンプレミス環境からのアクセスに関する設定については記載していません）。

![alt](https://github.com/takeucho/til/blob/main/images/fsx-ontap-nextcloud.png)

#### 対象読者
FSx for ONTAP、Amazon RDS、Amazon Elastic Load Balancing、Amazon ElastiCache、Nextcloud の知識を有している人。

#### この記事で説明しないこと
各手順の詳細は説明していません。

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
Snap Packageを使用してインストールした場合、NextCloudをインストールしているUbuntuへのcifs-utilsのインストールがうまくいかなかったため、FSx for ONTAPなどのSMBファイルサーバをマウントさせることはできませんでした。

## Nextcloud の標準ファイルディレクトリとして FSx for ONTAP の NFS 共有を使用する
今回、NextCloud は 2 台の EC2 で冗長化するため、標準コンテンツを格納する下記ファイルディレクトリとして FSx for ONTAP の NFS 共有を使用する設定を行います。

`/var/www/nextcloud/data/`

NFS マウントに必要なツールをインストールします。

`sudo apt install nfs-common`

元のファイルディレクトリの中身を退避させます。

`sudo mv /var/www/nextcloud/data/ /var/www/nextcloud/data_back`
`sudo mkdir /var/www/nextcloud/data/`

FSx for ONTAP を NFS マウントします。
FSx for ONTAP の NFS IP アドレス = 198.19.255.130 の場合

`sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 198.19.255.130:/vol1 /var/www/nextcloud/data/`

自動マウントの設定を行います。

`sudo echo "198.19.255.130:/vol1 /var/www/nextcloud/data/ nfs4 defaults 0 0" >> /etc/fstab`

## Amazon RDS の構築
上記で構築した Nextcloud サーバのデータベースを Amazon RDS に置きかえるため、下記ドキュメントを参照して Amazon RDS を構築します。

https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateDBInstance.html

Nextcloudをインストールした Ubuntu に Maria DB クライアントをインストールします。

`sudo apt install -y mariadb-client`

## Amazon Elastic Load Balancing の構築
Nextcloud サーバを冗長化するため、下記ドキュメントを参照して Amazon Elastic Load Balancing を構築します。
https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/create-application-load-balancer.html

## Amazon ElastiCache の構築
メモリキャッシュを利用して高速化するために、下記ドキュメントを参照して Amazon ElastiCache を構築します。
https://aws.amazon.com/jp/getting-started/hands-on/building-fast-session-caching-with-amazon-elasticache-for-redis/1/

## Nextcloud の設定
