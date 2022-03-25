# Amazon FSx for NetApp ONTAP と Nextcloud で実現するファイルサーバ兼クラウドストレージ環境

この記事では Amazon FSx for NetApp ONTAP と Nextcloud を使って、ファイルサーバ兼クラウドストレージとしてシームレスに使える環境構築の概要手順を紹介します。

## FSx for ONTAP の構築
下記ワークショップの AWS Cloud Formation テンプレートを使用することで、Active Directory 含めた FSx for ONTAP 環境を構築できます。
https://github.com/aws-samples/amazon-fsx-workshop/tree/master/netapp-ontap/JP

## Nextcloud Web サーバの構築
下記 Nextcloud のドキュメントを参照し、Ubuntu 20.04 LTS のAmazon EC2 インスタンスにNextcloudをインストールします。
https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html

## Amazon RDS の構築

## Amazon Elastic Load Balancing の構築

## Amazon ElastiCache の構築
