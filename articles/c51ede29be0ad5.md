---
title: "localstackのS3にデータを転送するときにエンドポイントでハマった話"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["localstack", "s3", "aws", "terraform", "docker"]
published: true
---
## はじめに
業務でAWS SDK for PHPを使ってアプリからS3にデータを転送する機能を実装しています。
ローカル環境での実装に少しハマってしまったので、それまでの道のりをまとめます。

## ざっくりいうと
ハマったのはS3にファイルをアップロードする際のエンドポイントの指定です。
コンテナやlocalstackの構成から、エンドポイントは`http://localstack:4566`とすべきでした。

## 環境
### コンテナ
ローカル環境では以下のようなコンテナ構成になっています。
- アプリ
- DB
- localstack

### terraform
また、localstackのS3はterraformで構築しています。
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws",
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"

  access_key = "test"
  secret_key = "test"

  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    dynamodb = "http://localhost:4566"
    S3       = "http://localhost:4566"
  }

  S3_use_path_style = true
}

resource "aws_S3_bucket" "example" {
    bucket = "test-bucket"
}
```

## S3へのアップロードを試す
S3へデータを転送するために、localstackを使った方法を確認しました。
https://docs.localstack.cloud/aws/integrations/aws-sdks/php/

```php
// Configuring S3 Client
$S3 = new Aws\S3\S3Client([
    'version' => '2006-03-01',
    'region' => 'us-east-1',
    // Enable 'use_path_style_endpoint' => true, if bucket name is non DNS compliant
    'use_path_style_endpoint' => true,
    'endpoint' => 'http://S3.localhost.localstack.cloud:4566',
]);
```

これで実装したところ、以下のようなエラーが

```bash
objectUpload:Error executing "PutObject" on "http://S3.localhost.localstack.cloud:4566/test-bucket/hoge/hoge.json"; AWS HTTP error: cURL error 7: Failed to connect to S3.localhost.localstack.cloud port 4566 after 0 ms: Could not connect to server (see https://curl.haxx.se/libcurl/c/libcurl-errors.html) for http://S3.localhost.localstack.cloud:4566/test-bucket/hoge/hoge.json
```

ここでgeminiにエラーの原因を聞いたところ、`S3.localhost.localstack.cloud`なんていうホスト名は存在しないらしい。
確かに、terraformで構築したときはそのようなエンドポイントを設定していなかった。


次に、エンドポイントを`localhost:4566`に設定してみた。
```php
$S3 = new Aws\S3\S3Client([
    'version' => '2006-03-01',
    'region' => 'us-east-1',
    // Enable 'use_path_style_endpoint' => true, if bucket name is non DNS compliant
    'use_path_style_endpoint' => true,
    'endpoint' => 'http://localhost:4566',
]);
```

その時のエラーがこちら

```php
objectUpload:Error executing "PutObject" on "http://localhost:4566/test-bucket/hoge/hoge.json"; AWS HTTP error: cURL error 7: Failed to connect to localhost port 4566 after 0 ms: Could not connect to server (see https://curl.haxx.se/libcurl/c/libcurl-errors.html) for http://localhost:4566/test-bucket/hoge/hoge.json
```

terraformでlocalhost:4566で構築しているのになぜ・・・？と思い、これもgeminiに質問してみたところ、納得の回答を得た。

```bash
  問題の本当の原因：Dockerコンテナのネットワーク

  Dockerコンテナの中から localhost にアクセスすると、それはコンテナ自身を指してしまいます。


  つまり、現在のPHPアプリケーションは、
   1. webコンテナの中で動いている。
   2. endpointにhttp://localhost:4566が指定されている。
   3. webコンテナ自身の4566番ポートに接続しようとする。
   4. webコンテナの4566番ポートでは何も動いていない。
   5. 結果、「接続できない (Could not connect to server)」というエラーになる。

  という流れになっています。

  解決策：コンテナ名で通信する

  Dockerコンテナ同士が通信する場合、localhostの代わりにコンテナ名をホスト名として使います。


  docker psの結果を見ると、LocalStackのコンテナ名は localstack です。

  したがって、PHPのS3クライアント設定のendpointを、以下のように変更してみてください。
```

アプリとlocalstackでコンテナを分かれているので、アプリが呼び出すエンドポイントは`localhost:4566`ではなく`localstack:4566`であるべきとのこと。

```php
$S3 = new Aws\S3\S3Client([
    'version' => '2006-03-01',
    'region' => 'us-east-1',
    // Enable 'use_path_style_endpoint' => true, if bucket name is non DNS compliant
    'use_path_style_endpoint' => true,
    'endpoint' => 'http://localhost:4566',
]);
```

エンドポイントを変更して挙動確認したところ、localstackのS3にオブジェクトが転送されていることが確認できた。


```bash
# コンテナ外で
❯ aws s3 ls s3://test-bucket --endpoint-url=http://localhost:4566 --profile=localstack --recursive
2025-11-10 16:43:40         17 hoge/hoge.json
```

## まとめ
- コンテナ内のlocalhostは自分自身を指してしまうため、他のコンテナへの接続には使えない。
- コンテナ間で通信するには、ホスト名として相手のコンテナ名（サービス名）を指定する。
