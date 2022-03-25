# Amazon FSx for NetApp ONTAP と Nextcloud で実現するファイルサーバ兼クラウドストレージ環境

この記事では Amazon FSx for NetApp ONTAP と Nextcloud を使って、ファイルサーバ兼クラウドストレージとしてシームレスに使える環境構築の概要手順と構築時の注意点を紹介します。
最終的な想定構成は以下となります（本記事ではオンプレミス環境からのアクセスに関する設定については記載していません）。

![alt](https://github.com/takeucho/til/blob/main/images/fsx-ontap-nextcloud.png)

## FSx for ONTAP の構築
下記ワークショップの AWS Cloud Formation テンプレートを使用することで、Active Directory 含めた FSx for ONTAP 環境を構築できます。
https://github.com/aws-samples/amazon-fsx-workshop/tree/master/netapp-ontap/JP

## Nextcloud サーバの構築
下記 Nextcloud のドキュメントを参照し、Ubuntu 20.04 LTS のAmazon EC2 インスタンスにNextcloudをインストールします。
https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html

## Amazon RDS の構築
上記で構築した Nextcloud サーバのデータベースを Amazon RDS に置きかえるため、下記ドキュメントを参照して Amazon RDS を構築します。
https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/CHAP_Tutorials.WebServerDB.CreateDBInstance.html

## Amazon Elastic Load Balancing の構築
Nextcloud サーバを冗長化するため、下記ドキュメントを参照して Amazon Elastic Load Balancing を構築します。
https://docs.aws.amazon.com/ja_jp/elasticloadbalancing/latest/application/create-application-load-balancer.html

## Amazon ElastiCache の構築
メモリキャッシュを利用して高速化するために、下記ドキュメントを参照して Amazon ElastiCache を構築します。
https://aws.amazon.com/jp/getting-started/hands-on/building-fast-session-caching-with-amazon-elasticache-for-redis/1/

## Nextcloud の設定
