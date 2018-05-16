# 概要
 **内容は全てリリース前のものであり、予告なく変更となる場合があります**

ちきゅうの公開APIの仕様について説明します。

 * プロトコルは全て「https」を利用。
 * リクエスト / レスポンスはどちらも「JSON」となる。
 * メソッドはPOSTのみとなる。
 * 認証方式により、「class0」「class1」「class2」に分けられる。
 * 毎秒10リクエストまで、1日5000リクエストまでの制限がある。

# APIのレベル

## class0
 * 認証なし、エンドポイントにリクエストを送信するだけで利用可能。

curlによるリクエスト送信例
```
curl -X POST -H 'Content-Type:application/json' -d '{"data": {"token_name":"name", "email":"email", "password":"password"}}' https://gateway.chikyu.mobi/dev/api/v2/open/session/token/create
```

## class1
 * HTTPヘッダに2つの認証用フィールド(APIキー)を追加し、エンドポイントにリクエストを送信する。
   * x-api-key
   * x-auth-key
 * 認証キーは、class-2のAPIとしてそれを生成するエンドポイントが存在する。
 * 生成の際は関連するロールのIDが必須となり、それによって呼び出し可能な処理が制御される。
 * 作成されたデータの作成者/変更者は「APIユーザー」となる。
   * ちきゅうの登録ユーザーとは関連付けられない。
 * APIキーの有効期限は設定されない(不要になったら削除)。
 * オプションとして、APIキーを利用可能なIPアドレスリストを指定できる。

curlによるリクエスト送信例
```
export API_KEY=AAAABBBBCCC
export AUTH_KEY=XXXXXXYYYYYYYZZZZZZZ
curl -X POST -H 'Content-Type:application/json' -H "x-api-key:$API_KEY" -H "x-auth-key:$AUTH_KEY" -d '{"data": {"page_index":0, "items_per_page":10}}' https://gateway.chikyu.mobi/dev/api/v2/public/entity/prospects/list
```

## class2
 * 「認証トークン」から「セッション」を生成し、それを利用して通信を行う。
   * メールアドレスとパスワードから、認証トークンを生成。
   * 認証トークンから、セッションを生成。
   * 生成したセッション情報を元に、AWSの提供する「署名バージョン4署名プロセス」に準拠するリクエスト署名を行って送信。
   * HTTPヘッダ(x-api-key)の付加も必要。
 * 認証トークン, セッションにはそれぞれ有効期限が設定される。
   * デフォルトでは、認証トークンが24時間, セッションが12時間。
   * 現状、セッションは認証基盤として利用するAWS Cognitoの制限により12時間以上に延ばすことができない。
   * 認証トークンに関しては、特に制限はない。
 * 認証トークン, セッションの生成を行うAPIはclass0のAPIとして提供される。
 * 作成されたデータの作成者/変更者は、認証トークンを取得したユーザーとなる。
   * ちきゅうに登録しているユーザーと関連付けられる。
 * 認証済リクエストの発行に関わる一連の処理は、「ちきゅうSDK」として提供される。
   * 認証トークンの発行 / 破棄
   * セッションの生成 / 破棄
   * 署名済みリクエストの発行
 * SDKは以下の言語に対応する。
   * Ruby
   * Java
   * JavaScript(jQuery)
   * Python
   * PHP

Class2はcurlで手軽にリクエストを送信することができません。

以下、Rubyの場合の例となります(詳しくはSDKのレポジトリを参照して下さい)。

```test.rb
require 'chikyu/sdk' # 事前にインスト−ルしておく。

# 2018/05/15現在、まだ本番環境が存在しないため、接続先の指定が必要。
Chikyu::Sdk::ApiConfig.mode = 'devdc'

# トークンを生成(一度生成したら値を保存しておくことで、長期間に渡り再利用可能)
token = Chikyu::Sdk::SecurityToken.create 'token_name', 'email', 'password'

# セッションを生成(再利用できるのは最大で12時間まで)
session = Chikyu::Sdk::Session.login(
  token_name: 'token_name',
  login_token: token[:login_token],
  login_secret_token: token[:login_secret_token]
)

invoker = Chikyu::Sdk::SecureResource.new session

# APIを呼び出し
p invoker.invoke(
  path: '/entity/companies/list', 
  data: {items_per_page: 10, page_index: 0}
)
```

# Webhook
「ワークフロー機能」にて、条件に合致する形のデータが作成/更新された際、任意のURLにそのデータを送信することが可能です。

# コレクション名について
 * APIドキュメント内で「{collection_name}」として表現されるプレースホルダに入力可能な値一覧

  日本語名称 | 英字名
  :---|:---
  見込客 | prospects 
  会社 | companies
  担当者 | customers
  商談 | business_discussions
  商品 | merchandises
  活動履歴 | activity_histories
  タスク | tasks
  商談(繰り返し計上) | repeat_recorded_business_discussions
  商談商品 | opportunity_merchandises

# API base URL
2018/05/14現在、テスト用のみ提供しています(本番リリース後には利用不可)。
## class0
https://gateway.chikyu.mobi/dev/api/v2/open/
## class1
https://gateway.chikyu.mobi/dev/api/v2/public/
## class2
https://gateway.chikyu.mobi/dev/api/v2/secure/

# APIリスト
http://dev-docs.chikyu.mobi/
