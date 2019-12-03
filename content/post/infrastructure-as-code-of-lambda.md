---
title: "lambdaのコード管理/Infrastructure as Code of Lambda"
date: 2018-12-15T14:32:03+09:00
draft: true
---

## はじめに

当記事はmeguro.dev#5 で発表した[lambdaのソース管理](https://speakerdeck.com/reizist/infrastructure-as-code-of-lambda)を更に初学者向けに詳細を書いたものです。

Infrastructure as Code という言葉があるように、資産のコード管理は非常に重要であることは
既に周知の通りです。
その中で、AWS Lambdaについてはコード管理に関して明確な方法というものはなくチームそれぞれのやり方で行っているのが現状だと思います。

ここでは2つの方法を紹介し、自分のチームでメインに使用しているapexについて更に掘り下げようと思います。

## SAM

まず、 [AWS Serverless Application Model(AWS SAM)](https://github.com/awslabs/serverless-application-model)
を紹介します。

名前の通りAWS公式のツールであり、CloudFormation拡張のサーバーレスフレームワークです。
CloudFormationと似たDSLでlambda本体及び関連リソースを定義でき、パッケージング/デプロイ共にCLIで統括的に提供されています。

公式だけあり、re:Invent2018で紹介された **Lambda Layerにも既に対応しています**。

```
aws cloudformation package --template-file template.yaml --output-template-file serverless-output.yaml --s3-bucket test-reizist

aws cloudformation deploy --template-file /Users/reizist/Desktop/lambda/serverless-output.yaml --stack-name sinatra --capabilities CAPABILITY_IAM
```

のように、まずSAMを定義したtemplateをs3にパッケージングし、その結果を使ってcloudformationでstack作成をする形になります。



## APEX

次に、[APEX](https://github.com/apex/apex)を紹介します。

こちらは非公式ではありますが、lambda自体のソース管理に加えterraform formatなインフラリソースを統合的に管理でき、デプロイ/実行/ログ確認を行えるツールとなっております。

Lambda Layerには現時点で非対応となっておりますが、[検討中のようです](https://github.com/apex/apex/issues/915)。


## 実例

re:Invent2018でLambdaのruby対応が発表されたので、apexを使ってruby及びrubyのweb framework, sinatraをlambdaで動かす実例を紹介します。

### pure ruby
まずはシンプルなケースでlambda functionのみのケースとしてrubyを使ってみます。

```
reizist ...apex test-lambda* $ tree . -L 3
.
├── functions
│   └── ruby_job
│       ├── function.prod.json
│       ├── lambda_function.rb
│       └── test.json
└── project.prod.json

2 directories, 4 files
```

ファイル構成はこのようになっています。
`lambda_function.rb` にlambda functionのソースを記述します。

```ruby
require 'json'

def lambda_handler(event:, context:)
    { statusCode: 200, body: JSON.generate(event) }
end

```

これをdeployするには以下のようなコマンドを実行します。

```sh
reizist ...apex test-lambda* $ apex deploy  --env prod ruby_job
   • config unchanged          env=prod function=ruby_job
   • updating function         env=prod function=ruby_job
   • updated alias current     env=prod function=ruby_job version=10
   • function updated          env=prod function=ruby_job name=reizist_ruby_job version=10
```

invokeコマンドで実行できます。

```sh
reizist ...apex test-lambda* $ apex invoke ruby_job --env prod
{"statusCode":200,"body":"{}"}
```

testパラメータの注入も簡単にできます。

```json
{
    "token": "test token"
}
```

```sh
reizist ...apex test-lambda* $ apex invoke ruby_job --env prod < functions/ruby_job/test.json
{"statusCode":200,"body":"{\"token\":\"test token\"}"}
```

実行ログを見る場合はlogコマンドを使用します。

```sh
reizist ...apex test-lambda* $ apex logs --env prod ruby_job
/aws/lambda/reizist_ruby_job START RequestId: d522052a-002f-11e9-9773-430855111917 Version: 11
/aws/lambda/reizist_ruby_job END RequestId: d522052a-002f-11e9-9773-430855111917
/aws/lambda/reizist_ruby_job REPORT RequestId: d522052a-002f-11e9-9773-430855111917	Duration: 47.87 ms	Billed Duration: 100 ms 	Memory Size: 128 MB	Max Memory Used: 18 MB
```

簡単ですね。

### sinatra

次にapi gatewayのリソース管理もapexで行うケースをsinatra例に紹介します。
尚アプリケーション実装としては [https://github.com/aws-samples/serverless-sinatra-sample](https://github.com/aws-samples/serverless-sinatra-sample)を使っています。

```sh
.
├── functions
│   └── sinatra_job
│       ├── Gemfile
│       ├── Gemfile.lock
│       ├── app
│       ├── function.prod.json
│       ├── lambda_function.rb
│       ├── test.json
│       └── vendor
├── infrastructure
│   ├── modules
│   │   └── sinatra_job
│   └── prod
│       ├── main.tf
│       └── variables.tf
└── project.prod.json

8 directories, 8 files
```

terraformのmodule機能を使い、インフラのリソースは全てmodule内に閉じ込めて `prod/main.tf` から参照する構成になっています。またGemfileで管理している依存ライブラリは `bundle install --path vendor/bundle` としlambda functionのソースと同ディレクトリに含めておく必要がある点に注意です。

こちらも同様に `apex deploy` した後api gatewayのendpointにwebブラウザでアクセスすると動作が確認できるはずです。

なお今回のソースは[githubに置いておきました](https://github.com/reizist/apex_lambda_sinatra)。


## 所感

公式だけあり、SAMは新機能への対応も早く最低限の機能は揃っています。とはいえlambdaのソースをパッケージング->デプロイするのに2段階踏まなくてはいけない点など個人的には少し面倒に感じました。
apexはソース管理の観点ではコマンド1つでデプロイでき、そのまま実行、ログ確認まで行えるので素早さはapexに利があるように感じています。

定義DSLでいうとSAMのDSLを別途理解する必要がある点は不便がありますが、小さい単位ではあとでDSLを見返すと直感的ではあるのかなと感じました。

例えば、今回sinatraを動かすにあたって定義したapi gatewayのリソースは[SAMの場合](https://github.com/aws-samples/serverless-sinatra-sample/blob/master/template.yaml)と[terraformの場合](https://github.com/reizist/apex_lambda_sinatra/blob/master/infrastructure/modules/sinatra_job/apigateway.tf)を比較すると、リソースの依存関係が（インデントのため）SAMのほうが見やすいように思います。

しかしながら規模が大きくなるにつれ、小さい単位に切り出してRef参照を行うなどのことをするとその利点もなくなるので、大規模での管理でいうとterraformが好みです。

チームがもともとインフラのソース管理にterraformを使っているのか、cloudformationを使っているのかでも変わってくるので状況にもよると思いますが、何かの参考になれば幸いです。








