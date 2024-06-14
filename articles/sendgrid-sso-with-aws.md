---
title: "AWS IAM Identity Centerを使ってSendGridにSSOログインする"
emoji: "🔒"
type: "tech"
topics: ["sendgrid","aws","sso"]
published: true
---

SendGridのSSOログイン機能（Proプランで利用可能）をAWS IAM Identity Center（旧AWS SSO）を使って実現する手順を紹介します。
SendGridのドキュメントにはAzure ADとOktaを使ったSSOの設定方法が記載されていますが、AWS IAM Identity Centerは記載されていなかったので参考になればと思います。

SendGridのドキュメントはこちら。
https://www.twilio.com/docs/sendgrid/ui/account-and-settings/sso

## AWS IAM Identity Centerでアプリケーションを作成

IAM Identity Centerのメニューからアプリケーションを選択して「アプリケーションの追加」をクリックします。
事前定義されたカタログの中にSendGridはないので、アプリケーションタイプに「SAML 2.0」を選択して次へ進みます。


「表示名」に適当にタイトルを入力します（ユーザーに表示される名前）。
![](https://storage.googleapis.com/zenn-user-upload/2799e8c420c9-20240614.png)


その下に各種URLが表示されています。これを次の手順で使います。
あと「IAM Identity Center 証明書」も後で使うのでダウンロードしておきます。

![](https://storage.googleapis.com/zenn-user-upload/0c04d60de9e6-20240614.png)

## SendGridのSSO設定

ここで一旦別タブでSendGridの管理画面を開いて、[Settings > SSO Settings](https://app.sendgrid.com/settings/sso)から設定を行います。

①まず下図のように「Name」に適当な名前をつけます。

②IdP Metadataという欄に先ほどIAM Identity Centerで表示されていたURLを入力します。
「SAML Issuer ID」と「Embed URL」の二つがありますが、どちらも入力するのは
```
https://portal.sso.REGION.amazonaws.com/saml/assertion/RANDOM_STRING
```
の形式のURLです（パスの途中に"assertion"が入っているもの）。

![](https://storage.googleapis.com/zenn-user-upload/40dcd656e97c-20240614.png)

「Twilio SendGrid Metadata」のURLはまた後で使います。

③証明書（Certificates）を追加します。
先ほどダウンロードしたIAM Identity Centerの証明書のpemファイルをテキストエディタで開いて、その中身をコピペします。

![](https://storage.googleapis.com/zenn-user-upload/019a183af53f-20240614.png)

④設定を保存して一旦SendGrid側の操作は完了です。

## IAM Identity Centerの設定（続き）

またIAM Identity Centerに戻り、先ほどのSendGrid MetadataのURLを設定画面下部の「アプリケーションメタデータ」の欄に入力します。2箇所ありますがどちらも同じです。

![](https://storage.googleapis.com/zenn-user-upload/064f96310ec7-20240614.png)

「送信」ボタンを押して設定を完了します。

作成したアプリケーションの設定画面に戻って、下の図の位置から「属性マッピングを編集」します。

![](https://storage.googleapis.com/zenn-user-upload/012d7230823f-20240614.png)

ここでは、SAML SP（=SendGrid）に送信されるSAMLアサーションの属性を指定できます。
以下のように、IAM Identity Centerのユーザー属性として`${user:email}`、形式として`emailAddress`を指定します。

![](https://storage.googleapis.com/zenn-user-upload/19dee0a98e2d-20240614.png)

SendGridはメールアドレスをユーザー識別子として使い、NameIDの形式として`emailAddress`を指定するよう[ドキュメントに記載](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/sso#troubleshooting)されています。
上記のように指定することで、IAM Identity Centerのユーザー属性のメールアドレスがSAMLアサーションに含まれるようになります。

ちなみに、IAM Identity Centerが持っているユーザー属性は以下に記載されています。
https://docs.aws.amazon.com/ja_jp/singlesignon/latest/userguide/attributemappingsconcept.html#supportedssoattributes

「変更の保存」ボタンを押せばAWS側の設定は完了です。

## SSO Teammateユーザーの作成
再度SendGridの管理画面に戻り、[Settings > Teammates](https://app.sendgrid.com/settings/teammates)からSSOユーザーを追加します。
右上の「Add Teammate」ボタンをクリックすると「Add password teammate」と「Add SSO teammate」の2つが表示されますが、ここでは「Add SSO teammate」を選択します。

IAM Identity Centerのユーザーのメールアドレスを入力してSSO Teammateを作成します。途中、アクセス可能な権限セット等も設定できます。
![](https://storage.googleapis.com/zenn-user-upload/03f2a1d859e9-20240614.png)

以上で設定は完了です。これでIAM Identity Centerのユーザーはログイン後にSendGridにSSOログインできるようになります 🎉