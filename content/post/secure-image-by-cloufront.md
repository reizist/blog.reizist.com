---
title: "secure-image-by-cloudfront"
date: 2018-12-04T05:23:43+09:00
draft: true
---

## はじめに

画像権限、管理していますか？
複数の組織をユーザーとして抱えるようなWebサービスにおいて、組織内のプライベートな情報として参照できる環境上で画像をアップロードをしたとしても、一度一意の画像URLを取得しまえばその後ログインせずともパブリックに画像参照できる、という状況の方が多いと思います。

今回はCloudFrontの機能を使って一定の条件でセキュアな環境を開発環境用に構築したので紹介いたします。

## 想定要件
具体的には下記の要件を満たします。

* 組織Aに所属するユーザーがアップロードした画像は、組織Aに所属する他のユーザーは全員参照でき、組織Aに所属しないユーザーは参照できない
* ユーザーは複数の組織に所属できる
* 複数組織に所属するユーザーは横断して画像の参照を行うことができる
* 画像のキャッシュは使いたい
* 画像のURLがバレても正当なアクセス権を持つ人間しか画像参照ができない


## 方針
まず第一に画像を得るためにリクエストパラメータに従い規定の画像レスポンスを返す、認証有り画像アプリケーションサービスを作るという方針があるかと思います。
しかしながら本件ではこれを見送りました。いくつか理由がありますが、現時点の想定環境下において管理コンポーネントを増やしたくない強い理由があったためです。
別のアプリケーションとして実装する判断はMicroservicesの観点では特別異常なことでもないように思いますが、今回は管理コンポーネントを増やしたくないという気持ちを優先してAWS, 具体的にはCloudFrontの機能でできないかを検討しました。

ちなみに本節の内容は [署名付きURLと署名付きCookieの選択](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/private-content-choosing-signed-urls-cookies.html)に詳しく記載があります。

### Signed URLs
セキュア環境を実現するためのCloudFrontの機能としてまず署名付きURLが挙げられます。
これは、**個別の画像に対し**都度個別に有効期限のあるパラメータ付きのURLを発行するという機能で、
画像URLが一意に固定できないことがキャッシュの観点で不効率であること、画像のURLがわかってしまえば誰でもアクセスできてしまうことなどから要件を満たさないので使いませんでした。

### Signed Cookies
次に署名付きCookieが挙げられます。基本的な機能は同じですが、
**複数の画像に対し**アクセス権のあるcookieを発行しこのcookieを用いて画像を参照することができるようになるもので、
画像URLが一意に固定されること、画像のURLがわかっても画像にアクセスするにはcookieが必要=正当なアクセス権のあるユーザーしかアクセスできないという点でSigned URLよりも要件を満たしそうです。

### Signed Cookiesを用いた場合の要件可能可否
さて、署名付きCookieについてもう少し深掘りをしてみます。
署名付きCookieは、有効期間などのパラメータを指定する方法として

* 規定ポリシー
* カスタムポリシー

の2通りあります。[有効期間以外のパラメータも変更する場合、カスタムポリシーを使います](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-cookies.html#private-content-choosing-canned-custom-cookies)。

```json
{
   "Statement": [
      {
         "Resource":"URL of the object",
         "Condition":{
            "DateLessThan":{"AWS:EpochTime":required ending date and time in Unix time format and UTC},
            "DateGreaterThan":{"AWS:EpochTime":optional beginning date and time in Unix time format and UTC},
            "IpAddress":{"AWS:SourceIp":"optional IP address"}
         }
      }
   ]
}
```

ここで重要なことは、`Statement` キーに対してハッシュ(連想配列)の配列を指定できるように見えますが、実際には**1つのステートメントのみを含めることができる** という点です。
また `Resource` にも

> 0 個以上の文字に一致するワイルドカード文字 (*)、または 1 つの文字に一致するワイルドカード文字 (?)

という制限があります。
[カスタムポリシーに関してはこちらに記載があります](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/private-content-setting-signed-cookie-custom-policy.html)。

ではこれによってどのように「ユーザーは複数の組織に所属でき、横断して画像の参照を行うことができる」という要件を実現するのでしょうか。
どういう意味かというと、複数組織に所属するユーザー1が複数組織の画像を一括で要求した場合、
`Statement`が1つしか指定できずに`Resource`で指定できるパスも正規表現が使えない以上、CloudFront単体で解決する手段はなさそうだ、ということです。 (`Resource`に正規表現使えるようにするとか`Statement`が複数指定できるようになってほしい...)

### 解決案
cookieのPath属性を用いてこれを解決することができます。
cookieは同名キーのものを複数保持することができます。

<a href="https://s3-ap-northeast-1.amazonaws.com/techblog.bucket/wp-content/uploads/2018/12/cookie_sequence.png" rel="shadowbox[images]" ><img src="https://s3-ap-northeast-1.amazonaws.com/techblog.bucket/wp-content/uploads/2018/12/cookie_sequence-295x300.png" alt="" width="295" height="300" class="alignnone size-medium wp-image-17782" /></a>

※ 同名キーのSet-Cookieの扱いは実装に依ることに加え、

> Even though the Set-Cookie header supports the Path attribute, the Path attribute does not provide any integrity protection because the user agent will accept an arbitrary Path attribute in a Set-Cookie header. For example, an HTTP response to a request for http://example.com/foo/bar can set a cookie with a Path attribute of "/qux". Consequently, servers SHOULD NOT both run mutually distrusting services on different paths of the same host and use cookies to store security-sensitive information.

と[RFC6265](https://httpwg.org/specs/rfc6265.html)に記載があるため十分に取扱いに注意したほうがよさそうですね。
本件においてはCloudFront側でcookieの正当性を担保するのでpath依存の攻撃は成功しないと判断しましたが、更にいい方法があったら知りたいです。


## 最終的なシーケンス図

* A, Bという組織に所属するユーザーの場合
<a href="https://s3-ap-northeast-1.amazonaws.com/techblog.bucket/wp-content/uploads/2018/12/sequence.png" rel="shadowbox[images]" ><img src="https://s3-ap-northeast-1.amazonaws.com/techblog.bucket/wp-content/uploads/2018/12/sequence-239x300.png" alt="" width="239" height="300" class="alignnone size-medium wp-image-17783" /></a>

## まとめ
複数の組織を取り扱うWebサービス上で扱う画像について、セキュア(=正しい権限を持つユーザーにのみ正しく画像参照できる状態になっている)な環境にするためにCloudFrontの署名付きCookieを使用しました。
署名付きCookieの挙動である複数リソースの制御に不都合があったので、cookieのpathの機能を使用して解決しました。

## 感想
セキュリティ周りに関して、普段から意識するのは難しいと感じました。
またCookieはpath依存ではなくLamda@Edgeなどを利用した代替案もないか検討できそうなので引き続きより良い方法を考えてみたいと思います。



