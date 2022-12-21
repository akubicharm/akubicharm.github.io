# Azure AutomateとAzure Container Instanceを使ってPostgresqlへのデータロードの実行


Azure Container Instanceは、単体でコンテナを簡単に実行することができるPaaSサービスです。
Web App For ContainersなどはWebアプリケーション用のPaaSサービスですが、Azure Container InstaneはHTTP(S)外のプロトコルでのアクセスや、バックグラウンドジョブなどにも利用することができます。

今回は、Azure Database for Postgresqlにデータをロードするためのスクリプトを実行するコンテナアプリをAzure Container Instanceにデプロイし、Azure Automateでスケジュール実行していきます。


## Azure Database for Postgresqlの準備

Azure Databadse for Postgresqlのデプロイは、[クイックスタート:Azure portalを使ってAzure Database for POstgreSQLサーバーを作成する](https://learn.microsoft.com/ja-jp/azure/postgresql/single-server/quickstart-create-server-database-portal)を参考にしてください。


psqlコマンドでの接続方法は、AzureポータルのAzure Database for Postgresqlの管理画面のメニューの設定セクションの「接続文字列」で確認することができます。

![Postgresql 接続文字列](images/pg_connstring.png)


## Azure Container Instanceの準備

シンプルにPostgreSQLにテーブルを作成して、データをインサートするスクリプトを実行するだけのシンプルなコンテナイメージを作成します。

用意するファイルは以下の3つ
* ./Dockerfile
* ./tool/init.sql
* ./tool/dataLoad.sh


``` init.sql
create table if not exists todo_item (
  id serial primary key,
  description varchar(128),
  title varchar(32),
  finished boolean
);

insert into todo_item (description, title, finished) values('desc', 'title', false);
```

``` 
#!/bin/sh
DBNAME=[DB名]
PASSWORD=[パスワード]

psql "host=pgakubicharm.postgres.database.azure.com port=5432 dbname=$DBNAME user=myadmin password=$PASSWORD sslmode=require" -f init.sql
```


```
FROM alpine:latest

RUN apk --no-cache add postgresql-client
RUN mkdir /work
COPY ./tool /work/.

WORKDIR /work

CMD ["./dataLoad.sh"]
```

ファイルを準備したら、Azure Container Registoryの機能を使って、コンテナイメージをビルドしAzure Container Registryに登録します。

```
ACR_NAME=myacr
az acr build -t psql/dataload:automate -r $ACR_NAME --platform linux .
```
