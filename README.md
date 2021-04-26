# 概要
 ちきゅうの公開APIの仕様について説明します。

 * プロトコルは全て「https」を利用しています。
 * リクエスト / レスポンスはどちらも「JSON」となります。
 * メソッドはPOSTのみとなります。
 * 認証方式により、「class0」「class1」「class2」に分けられます。
 * 契約プランにより、呼び出し回数に制限が設定されています。
 * 
# APIのレベル

## class0
 * 認証なし、エンドポイントにリクエストを送信するだけで利用可能です。

curlによるリクエスト送信例
```
curl -X POST -H 'Content-Type:application/json' -d '{"data": {"token_name":"name", "email":"email", "password":"password"}}' https://endpoint.chikyu.net/api/v2/open/session/token/create
```

## class1
 * HTTPヘッダに2つの認証用フィールド(APIキー)を追加し、エンドポイントにリクエストを送信します。
   * x-api-key
   * x-auth-key
 * 認証キーは別のAPIを経由し、事前に生成しておきます。
   * キーを生成するAPIは、後述の「class2」のAPIを利用します。
 * 生成の際は関連するロールのIDが必須となり、それによって呼び出し可能なメソッドに制限がかかります。
 * 作成されたデータの作成者/変更者は「APIユーザー」となります。
   * ちきゅうに登録されているユーザーとは関連付けられません。
 * APIキーの有効期限は設定されません(不要になったら削除を行ってください)。
 * オプションとして、APIキーを利用可能なIPアドレスリストを指定できます。

curlによるリクエスト送信例
```
export API_KEY=AAAABBBBCCC
export AUTH_KEY=XXXXXXYYYYYYYZZZZZZZ
curl -X POST -H 'Content-Type:application/json' -H "x-api-key:$API_KEY" -H "x-auth-key:$AUTH_KEY" -d '{"data": {"page_index":0, "items_per_page":10}}' https://endpoint.chikyu.net/api/v2/public/entity/prospects/list
```

## class2 (SDK導入の方はここをご確認ください。)
 * 「認証トークン」から「セッション」を生成し、それを利用して通信を行います。
   * メールアドレスとパスワードから、認証トークンを生成します。
   * 認証トークンから、セッションを生成します。
   * 生成したセッション情報を元に、AWSの提供する「署名バージョン4署名プロセス」に準拠するリクエスト署名を行って送信します。
   * HTTPヘッダ(x-api-key)の付加も必要です。
 * 認証トークン, セッションにはそれぞれ有効期限が設定されます。
   * デフォルトでは、認証トークンが24時間, セッションが12時間となります。
   * 現状、セッションは認証基盤として利用するAWS Cognitoの制限により12時間以上に延ばすことはできません。
   * 認証トークンに関しては、無期限に有効とすることも可能です。
 * 認証トークン, セッションの生成を行うAPIはclass0(認証なし)のAPIとして提供されます。
 * 作成されたデータの作成者/変更者は、認証トークンを取得したユーザーとなります。
   * ちきゅうに登録しているユーザーと関連付けられます。
 * 認証済リクエストの発行に関わる一連の処理は、「ちきゅうSDK」として提供されます。
   * 認証トークンの発行 / 破棄
   * セッションの生成 / 破棄
   * 署名済みリクエストの発行
 * SDKは以下の言語に対応しております。
   * NodeJS (推奨)
   * JavaScript(jQuery)
   * Python
 * 以下のSDKは存在はしますが、古くなっており、現在は非推奨となっております。
   * Java
   * PHP
   * Ruby

Class2はcurlで手軽にリクエストを送信することができません。

以下、NodeJSの場合の例となります(詳しくはSDKのレポジトリを参照して下さい)。

get_login.js（ログイン情報を使用してトークンを生成する。）
```
const chikyu = require('./lib/chikyu');

// ここにログイン用のトークンネームをユーザー側で好きな文字列を設定します。
// 以下は例なのでそのまま使用しないでください。
token_name = 'nodejssdk_login001';
// ここにちきゅうにログインできるログインユーザーID(メールアドレス)
email = 'こちらを設定してください';
// その時のパスワード
password = 'こちらを設定してください';

// コンソールなどで、「node get_login.js」と、nodeで、このファイルを実行することにより、
// 以下が実行されて、ログインするためのtokenとsecret_tokenがリターンされますので、
// 上記に設定したtoken_nameと、リターンされたtokenとsecret_tokenを設定してご使用ください。
// 別ファイルのtest.jsに、トークン設定例を示します。
chikyu.token.create(token_name, email, password).then((r) => {
  console.log(r);
}).catch((e) => {
  console.log(e);
});
```

test.js（トークンを設定してログインしてAPIを実行する例。）
```
const chikyu = require('./lib/chikyu');

// トークンを一度生成したら、以下のように値を保存しておくことで、長期間に渡りログイン時に再利用可能

// ここにログイン用のトークンネームをユーザー側で好きな文字列を設定します。
// ここに、get_login.jsに設定したtoken_nameをセットしてください。
// 以下は例なのでそのまま使用しないでください。
token_name = 'nodejssdk_login001';
// ここに、get_login.jsでリターンされたtokenをセットしてください。
token = 'ここにget_login.jsでリターンされたtokenをセット';
// ここに、get_login.jsでリターンされたsecret_tokenをセットしてくださいをセットしてください。
secret_token = 'ここにget_login.jsでリターンされたsecret_tokenをセットしてください';

// セッションを生成(再利用できるのは最大で12時間まで)

// 上記で、設定された、token_name, token, secret_tokenの３つによって、ログインができます。
chikyu.session.login(token_name, token, secret_token).then((r) => {

  // ログイン後に、APIの実行

  // 以下はログイン後に、一覧取得の基本APIの「/entity/ここに取得したいオブジェクト名/list」に、
  // 商談(business_discussions)をセットして、商談の一覧Listを取得した例です。
  // ページ設定は必須です。page_index→0からスタートで何ページ目からか。items_per_page→１ページの表示件数。(Maxは500)
  return chikyu.secure.invoke('/entity/business_discussions/list', {'items_per_page': 10, 'page_index': 0}).then((d) => {
    console.log(d);
  });
}).catch((e) => {
  console.log(e);
});
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
  カスタムオブジェクト | カスタムオブジェクト作成時に設定したオブジェクトキー

# API base URL
## class0
https://endpoint.chikyu.net/api/v2/open/
## class1
https://endpoint.chikyu.net/api/v2/public/
## class2
https://endpoint.chikyu.net/api/v2/secure/

# APIリスト
ちきゅう内にあるチャットツールからCSにお問い合わせください。
