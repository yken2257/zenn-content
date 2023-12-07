---
title: "Gmailの新スパム規制対応全部書く"
emoji: "📩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["email", "Gmail", "DKIM", "SPF", "DMARC"]
published: true
---

2023年10月3日、Googleはスパム対策強化のため、Gmailへ送るメールが満たすべき条件を2024年2月から厳しくすると[発表](https://blog.google/products/gmail/gmail-security-authentication-spam-protection/)しました。また米国Yahoo!も、2024年第一四半期から同様の対策を行うと[発表](https://blog.postmaster.yahooinc.com/post/730172167494483968/more-secure-less-spam)しています。端的に言えば、この条件を満たさないと宛先にメールが届かなくなるという影響の大きな変更です。

この記事では、Gmailや米国Yahoo!の規制強化への対応方法を解説します。ただし米国Yahoo!にメールを送る人は多くないと思うので、フォーカスはGmail寄りです。また、メール配信サービス（海外だとSendGridやAmazon SES、国産だとblastengine等）の利用者寄りの視点になってます。

12月に入っていくつかアップデートがあり、まだ更新される可能性もありそうです。最新の情報は[英語版のガイドライン](https://support.google.com/mail/answer/81126?hl=en)や[FAQ](https://support.google.com/a/answer/14229414?hl=en)を見てください（日本語版の更新は英語版より遅れるようなので）。

:::message
筆者はGoogleやYahoo!の人間ではなく、この記事の内容は筆者の個人的な見解であることにご留意ください。新要件の詳細やメールの仕様に関してお気づきの点があれば、ご指摘いただければと思います。
:::

## 先にまとめ
最初に、Gmailの新要件の各項目について、状況を確認する方法と取るべき対策を表でまとめます。

| 要件 | 確認方法 | 対応方法 |
| --- | --- | --- |
| **SPF、DKIM、DMARC** | **メールヘッダ、チェックツール、DMARCレポートによる確認** | **DNSレコードの編集が必要**|
| **Reverse DNS** | **メールヘッダやチェックツールによる確認** | メール配信サービスを使っていれば自動で設定されるはず |
| TLS接続 | メールのToの左に赤い鍵マークがないか | メール配信サービスを使っていれば問題ないはず |
| **迷惑メール率** | **Google Postmaster Tools** | **0.3%を超えるなら適切な対策を（後述）** |
| RFC5322への準拠 | チェックツールを使う等 | メール配信サービスを使っていれば問題ないはず |
| 送信元Fromドメイン | メールの送信元を見る | `gmail.com`から送らない |
| ARCヘッダ | メールヘッダを見る | （メールを再配送せず、直接送信するなら関係ない要件） |
| **マーケティングメールのone-click unsubscribe** | **メールの「購読解除」表示やメールヘッダを見る** | **ヘッダを追加する** |

太字の項目が特に大事だと思います。
特に、SendGridやAmazon SES等のサービスにメール運用を任せている場合は、最低限対応が必要なのは2つだけだと思います。

- **DMARCを設定しよう！**
- **Google Postmaster Toolsを設定して迷惑メール率をウォッチしよう！**

自社サービスのドメインにDMARCレコードは設定されているでしょうか？
まずは`_dmarc.example.jp`のTXTレコードを[ここで調べて](https://toolbox.googleapps.com/apps/dig/#TXT/)ください（自社ドメインを`example.jp`とした場合）。

```
"v=DMARC1; p=none; rua=mailto:dmarc-report@example.jp"
```

みたいなのがあればひとまずOKです。なければ、設定しましょう。

なお「SendGridやAmazon SES等のメール配信サービスを使っていれば、勝手に設定してくれるんでしょ？」と期待している方は、そうではないのでぜひご確認ください。

## Googleの新スパム対策の概要
[Gmailのガイドライン](https://support.google.com/mail/answer/81126)によれば、Gmailアカウントに対し1日当たり送信するメールが約5,000を超えるかどうかで基準が異なります。ガイドラインに箇条書きで書かれている要件をここにも書いておきます。

### Gmailアカウントに送信するメールがおよそ5,000通/日を下回る場合
1. SPFとDKIM認証のいずれかを設定すること
2. Reverse DNS（"逆引きDNS"）を設定すること
3. 送信時にTLS接続を行うこと（※2023/12/6に追加された要件）
3. 迷惑メール報告率を0.1%未満に抑えること
4. メールの形式がRFC5322に準拠していること
5. 送信元Fromドメインとして`gmail.com`を使用しないこと
6. メールを再配送する場合はARCヘッダを付与すること

### Gmailアカウントに送信するメールがおよそ5,000通/日を超える場合
1. SPFとDKIM認証を**ともに**設定すること
2. Reverse DNS（"逆引きDNS"）を設定すること
3. 送信時にTLS接続を行うこと（※2023/12/6に追加された要件）
3. 迷惑メール報告率を0.1%未満に抑えること
4. メールの形式がRFC5322に準拠していること
5. 送信元Fromドメインとして`gmail.com`を使用しないこと
6. メールを再配送する場合はARCヘッダを付与すること
7. DMARC認証を設定すること
8. ヘッダFromドメインがSPFのドメインまたはDKIMのドメインと一致すること（これはDMARC認証の成功に必要）
9. マーケティングメールや受信者が配信を登録したメールは、one-click unsubscribeに対応し、メール本文にunsubscribeリンクを記載すること

5,000通以上では、1の条件が厳しいのと、8以降が追加されています。
[以後の章](#googleの新要件への対応)でそれぞれの要件について解説します。

### 5,000通の数え方
また、「5,000通」の定義が気になると思うのでここで公開情報をまとめておきます。

#### 通数の線引き
Gmailの[FAQページ](https://support.google.com/a/answer/14229414?hl=en)には"close to 5,000 or more"と表記されているので、4,999通に抑えればOK、ということではないです。5,000という数字にこだわりすぎないほうがいいと思います。
また、一回でも約5,000通/日を超えたらそれ以降は要件を満たしてね、とも取れる言い方をしています。
> Senders who have sent close to 5000 messages in a 24-period 1 or more times are considered bulk senders.

#### どの宛先に送るか
`gmail.com`ドメイン宛に何通メール送信しているかがカウントされます。`gmail.com`ドメイン以外の宛先に送るメールはカウントされません。
:::message alert
送信先の企業や団体がGoogle Workspaceを使っている場合は`gmail.com`ドメインではないので、カウントに含まれません。発表当初はGoogle Workspaceも含まれていたのですが、後に修正されたようです（2023/12/7現在英語版のガイドラインは更新されているが、日本語版はまだのよう）。
:::
#### どこから送るか
ヘッダFromドメイン（受信者が目にする送信元ドメイン）の単位でカウントされるとFAQページに[記載](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cwhat-is-a-bulk-sender:~:text=with%20the%20same%20domain%20in%20the%20From%3A%20header)されています。送信元IPアドレスを分けてもヘッダFromドメインが同じなら、同じ送信者から送信したことになりそうです。

## 送信ドメイン認証技術のおさらい

今回の要件はこれと言って真新しいものはなく、これまでずっと言われてきたメール送信ベストプラクティスを守ることが求められます。しかし、それを理解するにはメールの送信ドメイン認証技術を知っておく必要があります。最初に、その基本的なところを簡単におさらいしておきます。
:::message
SPF/DKIM/DMARCを知ってる方はこの章は飛ばしてください。
:::

### 2つの送信元From
メールに書かれる送信元Fromドメインには2種類あるのを知っていますか？
電子メールの構造は封筒と便箋によく似ています。封筒には、郵便配達員さんが見るための宛先住所と、不備があったときに差し戻す差出人住所が書かれていますね。一方で相手が実際に読む便箋の中にも、宛先や差出人の名前を書くのが普通でしょう。つまり、封筒と便箋の両方に宛先と差出人が書かれます。

この差出人ですが、嘘の情報を書いても届きます。封筒は宛先さえ正しければ、差出人の住所が正しいかどうかは郵便配達員さんは確認しません。便箋に書かれた差出人の名前も、実際にはその人の名前でなくても届きます。

| | 封筒の差出人 | 便箋の差出人 |
| --- | --- | --- |
| 見る人 | 届ける人（郵便配達員） | 届けられた人（宛先） |
| 書く目的 | 不備があったときに差し戻すため | 誰からの手紙か確認するため |
| 詐称できるか | 可能（宛先住所があれば、でたらめでも届く） | 可能（便箋に何を書くかは自由） |

![封筒と便箋のイメージ](https://storage.googleapis.com/zenn-user-upload/d0e80daa1b3b-20231126.png  =450x)
*封筒と便箋のイメージ（画像はChatGPTで生成しました）*

同様に、電子メールにも2種類の差出人（送信元From）があります。

1つ目は「エンベロープFrom」と呼ばれるもので、封筒の差出人に相当します。電子メールを届ける人＝メールサーバが確認するためのもので、普通は受信者個人は見ることはありません。2つ目は「ヘッダFrom」と呼ばれるもので、便箋の差出人に相当します。受信ボックスでメールを開くと表示されるやつです。これは単に表示名としての差出人です。表示名としての差出人は、メールサーバが返送処理するための差出人（エンベロープFrom）とは別個に存在するのです。

| | エンベロープFrom | ヘッダFrom |
| --- | --- | --- |
| 見る人 | 届ける人（メールサーバ） | 届けられた人（宛先） |
| 書く目的 | 不備があったときに差し戻すため | 誰からの手紙か確認するため |
| 詐称できるか | 可能（宛先住所があれば、でたらめでも届く） | 可能（何を書くかは自由） |

![](https://storage.googleapis.com/zenn-user-upload/6bf8ad9f32aa-20231208.png)

表の内容はリアル世界のケースと全く一緒です。
便箋に何を書くか自由なのと一緒で、電子メールの仕組み上、ヘッダFromはどんな内容も書けます。なので、実際は`narisumashi.example`ドメインのメールサーバなのに、ヘッダFromを`example.jp`と書いたメールを送るのは容易です。受信者はふつうヘッダFromしか見ないので、`narisumashi.example`から送られていることは知る由もありません。

このように電子メールは構造上なりすましが容易であるため、以下で述べる送信ドメイン認証技術が考案されてきました。

### 送信ドメイン認証技術 1. SPF

SPFは、メール送信元IPアドレスが、メール送信元ドメインの管理者によって認められているかどうかを確認する仕組みです。受信メールサーバはメールを受け取ると、そのエンベロープFromドメインのTXTレコードを見にいきます。エンベロープFromドメインのTXTレコードには、どのIPアドレスからメールを送るかが書いてあります。メールのヘッダに記載されている送信元IPアドレスが、送信元として指定してあるIPアドレスと一致すれば、SPF認証は成功します。リアルの世界で大雑把に例えるなら、封筒の差出人の名前（エンベロープFrom）を見て、その人が実際に住んでいる住所（送信元IPアドレス）からメールが届いたかどうかを判断するような具合です[^1]。

![](https://storage.googleapis.com/zenn-user-upload/6ab535e3fe89-20231208.png =500x)
*SPFの概略図*

[^1]: 簡単のためにこう表現しましたが、厳密に言えば「実際に住んでいる住所」という表現は語弊があります。エンベロープFromドメインのAレコードのIPアドレスから送る必要はなく、また送信元のIPアドレスは複数指定できます。差出人の名前ごとにどの住所から手紙を送るかのリストがあって、そのリストに書かれている住所から送る必要がある、という感じ？

### 送信ドメイン認証技術 2. DKIM

DKIMは公開鍵暗号を使ってメールの改ざんの有無を確認する仕組みです。送信側は、メールのヘッダのいくつか（Subject等）と本文をくっつけた文字列をハッシュ化し、秘密鍵で暗号化してメールヘッダに電子署名を記載します。受信側は、メールに付与されている電子署名を、送信元ドメインのTXTレコードに登録されている公開鍵で復号します。こうして取り出したハッシュ値が、送られてきたメールのヘッダと本文をハッシュ化したものと一致すれば、メールに改ざんがないことが確認できます。
注意点として、公開鍵をおいておくDKIM用のドメインは、エンベロープFromやヘッダFromとは別のものでも構いません。このドメインは電子署名を記載するDKIM-Signatureというヘッダの中で指定されています。

![](https://storage.googleapis.com/zenn-user-upload/96762df4d8a9-20231208.png =500x)
*DKIMの概略図*

SPFもDKIMもDNS問い合わせが発生するため、DNSレコードを管理できる人＝ドメインを管理する人がそのメールの確かさを保証していることになります。

### 送信ドメイン認証技術 3. DMARC

#### SPF/DKIMの課題
SPFとDKIMはそれぞれ、エンベロープFromドメインとDKIM-Signatureヘッダに記載されたドメインを起点に認証を行います。しかしこれでは、なりすましメールを完全には防止できません。なぜなら、受信者が目にするのはヘッダFromドメイン（便箋の差出人）であり、SPF/DKIM認証で使う（封筒の）ドメインはふつう意識されないからです。SPF/DKIMの規定上、認証に使うドメインをヘッダFromドメインに一致させる必要はないので、ヘッダFromドメインを`example.jp`にしておいて、SPFとDKIMを`narisumashi.example`で認証できてしまい、"セキュアな"なりすましメールが送れてしまいます。

#### DMARCの仕組み
それに対してDMARCはヘッダFromドメインが起点です。SPFとDKIMは送信するメール一通一通について正当性を担保する仕組みでしたが、DMARCは少し発想が異なり、自社ドメイン`example.jp`について、なりすましメールに対するポリシーを宣言する仕組みです。

具体的には、`example.jp`のDMARCレコードを設定することで、ヘッダFromドメインが`example.jp`になっているメールのうち、SPFとDKIM認証が失敗したメールをどう処理するか、受信メールサーバに指示することができます。
DMARCレコードは、`_dmarc.example.jp`のTXTレコードに以下のように記載します。

```
v=DMARC1; p=none; rua=mailto:dmarc-report@example.jp
```

##### DMARCポリシー
上のレコードの`p=none`の部分はDMARCポリシーと呼ばれるもので、DMARCのチェックに失敗したメールをどう処理するかの指示です。以下の3種類があります。

- `none` (指示なし): 特に決まった処理を要求しない
- `quarantine` (隔離): 疑わしいものとして処理するよう要求する
- `reject` (拒否): 拒否するよう要求する

メールクライアントは、ヘッダFromドメインのポリシーを確認し、`quarantine`が宣言されていたら、DMARC認証に失敗したメールを迷惑メールフォルダに振り分けます。`reject`の場合は、受信ボックスに届けずに弾きます。

##### アライメント
DMARCの要件として、SPFとDKIMのアライメントがあります。これは、ヘッダFromドメインがSPFまたはDKIMのドメインと一致することを要求するものです。これにより、ヘッダFromドメインを`example.jp`にして、SPFとDKIMを`narisumashi.example`で認証した場合は、DMARC認証は失敗します。

##### DMARCレポート
DMARCレコードには、`rua`というフィールドを設定できます。これはDMARCレポートの送信先メールアドレスを指定します。DMARCレポートはメールクライアント側からDMARC認証の成功や失敗の状況が定期的に送信される仕組みです。これにより、なりすましメールが送信されていないか、自社のメールがDMARCに失敗していないかなどが確認できます。

はじめは`p=none`のポリシーに設定してレポートで状況を確認し、正しいメールがきちんとDMARC認証に成功するようになったら`p=quarantine`や`p=reject`に変更していくのがセオリーです。

### 参考資料
SPF/DKIM/DMARCの詳細については、SendGridのブログ記事や「なしすまし対策ポータル ナリタイ」が詳しいです。
https://sendgrid.kke.co.jp/blog/?p=10121
https://www.naritai.jp/

## Googleの新要件への対応
前置きが長くなりましたが、ここからGoogleの新要件への対応方法を述べていきます。
見出しの★は、各要件の緊急度、重要度の目安（筆者の独断と偏見かつメール送信SaaS利用者の視点）です。
:::message
以下では5,000通の基準で場合分けせず、厳しい方の条件を満たす方法を述べます。
:::

### 送信ドメイン認証【★★★★★】
上記の要件の箇条書き1, 8, 9番目がSPF/DKIM/DMARCへの対応の要件です。
#### SPFとDKIM
SPFとDKIMは最も基本的な送信ドメイン認証で、個人の観測範囲では大多数のメールが設定しているように思います。
##### 認証成功（PASS/FAIL）の確認方法
自分が送っているメールの状況を知るには、Gmailで受け取ってみて、メールのヘッダを見るのが手っ取り早いでしょう。Gmailのヘッダは右側の三点リーダーから「メッセージのソースを表示」を選択すると見ることができます。

![](https://storage.googleapis.com/zenn-user-upload/42c2a95958e9-20231116.png  =300x)

「メッセージのソースを表示」では、メールの基本的な情報と共にSPF/DKIM/DMARCの認証結果が表示されます。左のメールはSPFはPASSしていますが、DKIMとDMARCはそもそも設定されていません。右のメールはSPF/DKIM/DMARC全ての認証がPASSしています。

![](https://storage.googleapis.com/zenn-user-upload/ca79f17193d4-20231116.png)

ちなみに、メールヘッダを見るのが億劫という方は、専用のチェックツールを使うのがおすすめです。SPF/DKIM/DMARCの認証結果だけでなく、Reverse DNS、メールのフォーマット、内容（迷惑メール度）等も一緒に確認できます。SendGridのブログでいくつか紹介されているので貼っておきます。どれも、サービスにメールを送信してチェックしてもらう形式です。
https://sendgrid.kke.co.jp/blog/?p=5190
https://sendgrid.kke.co.jp/blog/?p=14778
https://sendgrid.kke.co.jp/blog/?p=8210

##### DMARCのアライメント
SPF/DKIMに関しては、単に認証を通すだけではなく、DMARCのアライメントの条件を満たす必要があります（条件の9番目）。以下のいずれかを満たさないといけません。
- DKIMのドメイン（「DKIM: 'PASS'」の欄の横に記載されているドメイン）とヘッダFromドメイン（「From:」の欄のドメイン）が一致すること
- SPFのドメイン（`Return-Path`ヘッダに指定されているドメイン）とヘッダFromドメインが一致すること

これは、「メッセージのソースを表示」の上部の表示と、ソースの中にあるReturn-Pathヘッダのドメインを見ることで確認できます。
以下はどちらも満たしているAmazon Payのメールです。

![](https://storage.googleapis.com/zenn-user-upload/ff68b02299ed-20231126.png)
*ヘッダFromドメイン、SPF、DKIMのドメインはいずれもamazon.com*

あるいは、メールを開いた時のToの横の「▼」から確認することもできます。「送信元」がSPFのドメイン（エンベロープFrom）で、「署名元」がDKIMのドメイン（DKIM-Signatureヘッダのドメイン）です。
![](https://storage.googleapis.com/zenn-user-upload/a974011ce06c-20231206.png)

:::message alert
DMARCのデフォルトではサブドメインの違いは許されます。なおサブドメインも含めて厳密に一致することを送信者側が指定することも可能です。
:::

##### 対応方法
手元のメールで以下のケースに該当したら、修正を検討する必要があります。
- SPFとDKIMのいずれかの項目が欠けていた
- SPFとDKIMのいずれかがFAILしていた
- SPFとDKIMは共にPASSしていたが、どちらのドメインも、ヘッダFromドメインと一致していなかった

SendGridやAmazon SESなどのサービスを使っている場合、この要件はサービス側の設定でクリアできるはずなので、詳細な対応方法は割愛します。

なお、手元にあるメール一通がPASSしていても安心はできません。送信元Fromとして複数のサブドメインを使っていたり、メール送信にさまざまな外部ツール[^2]を使っていたりすると、全てのメールの状況をチェックするのは難しいです。DMARCレコードを設定して、SPF/DKIM認証の状況をruaレポートで確認するのが良いでしょう。

また、仕組み上、正しいメールでもSPF/DKIMはしばしば失敗します[^3]。送信数が5,000通/日以下の場合はSPF/DKIMのどちらかだけで良いことになっていますが、片方失敗した場合に備え、できれば両方設定＋DMARCも設定してレポーティングすることをおすすめします。GmailのFAQにも、アライメント含め両方設定するのが良い、いずれは両方設定することが要件になると[述べられて](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cwhat-is-a-bulk-sender%2Cwhat-happens-if-senders-dont-meet-the-requirements-in-the-sender-guidelines%2Cif-messages-are-rejected-because-they-dont-meet-the-sender-guidelines-do-you-send-an-error-message-or-other-alert%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined%2Cwhat-is-the-dmarc-alignment-requirement-for-bulk-senders:~:text=It%E2%80%99s%20likely%20that%20DMARC%20alignment%20with%20both%20SPF%20and%20DKIM%20will%20eventually%20be%20a%20sender%20requirement)います。


[^2]: SendGridやAmazon SESといったザ・メール配信サービスだけでなく、SalesforceやHubSpot等のCRMとか、Auth0（認証）とかZendesk（ヘルプデスク）とかkintoneとか、メールを送る機能を持つサービスはたくさんあります。自社で使っているサービス、全部把握していますか？
[^3]: メールが転送される等で送信元IPアドレスが変わるとSPF認証が失敗する、メーリングリスト等でメールのヘッダが改変されるとDKIM認証が失敗する、などそれぞれ弱点があります。

#### DMARC
##### 要件の解釈
[Gmailガイドライン](https://support.google.com/mail/answer/81126?hl=en#zippy=%2Crequirements-for-sending-or-more-messages-per-day)においてDMARCは箇条書きの7番目に述べられています。

> Set up DMARC email authentication for your sending domain. Your DMARC enforcement policy can be set to **none**.

一文目は「DMARC認証をPASSせよ」というよりは「DMARCレコードを設定せよ」という意味だと思います。なぜなら、要件箇条書きの1つ目の「SPFとDKIMを設定せよ」と、8番目の「ヘッダFromドメインをSPFまたはDKIMのドメインと一致させよ」を満たせば必然とDMARC認証がPASSするからです。結局、総合的には「DMARCレコードを設定し、かつPASSせよ」という要件になります[^4]。

「ポリシーは`none`にしていいよ」という2文目も解釈がややこしいです。まず`p=none`のよくある誤解として、「何もしない」＝「問答無用でメールを届けてくれ」という意味ではありません。特定のアクションを要求しないというポリシーなので、実際にメールを届けるかどうかは受信メールサーバの判断に委ねられます[^5]。SPFやDKIMも通っていない明らかな不正メールが`p=none`で届くようになるわけではありません。
文字面は「DMARCレコードを設定してね。ポリシーは`none`でいいよ」ですが、他の要件でDMARC認証PASSを要求しているので、実質DMARCをPASSしないと受信ボックスに届かないと考えた方が良さそうです。「送信側のポリシーは`none`でもいいよ（でもGmailの受信ポリシーは一律`quarantine`以上相当だからね）」という裏の意味があります。

少しややこしいので以上まとめると、

- `p=none`のポリシーは、メールを受信ボックスに届けさせる強制力はない
- メールを受信ボックスに届けるかどうかは受信側のポリシーで決めて良い
- Gmailは今回の規制強化で、DMARC認証PASS相当の条件を満たすことを要求している

ので、`p=none`で設定したとしても、Gmailでは`p=quarantine`相当のポリシーが適用されると考えられます。[^a]

[^4]: 箇条書きの1番目と9番目（「SPFとDKIMを設定する」「アライメントを一致する」）を満たせばDMARCはPASSするので、8番目の条件は実質いらないことになります。わざわざ8番目の要件があるのは、ちゃんとDMARCレコードを設定しろという意図かなと思います。つまり、DMARC認証PASSの条件を満たしているけどDMARCレコードがないドメインのメールは、弾かれる可能性があるということになります。

[^a]: [GmailのFAQ](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined:~:text=If%20messages%20fail%20DMARC%20because%20of%20authentication%20or%20alignment%20issues%2C%20the%20enforcement%20defined%20in%20the%20sending%20domain%E2%80%99s%20DMARC%20policy%20generally%20applies)には`If messages fail DMARC because of authentication or alignment issues, the enforcement defined in the sending domain’s DMARC policy generally applies`と書いてありますが、これは`p=none`なら受信トレイに素通りで届くという意味ではないと思います。

##### やるべきこと
自社ドメインにDMARCレコードがない場合、`p=none`ポリシーでDMARCレコードを設定しましょう。Gmailでは`p=quarantine`相当なのだから`p=quarantine`にしちゃえば良いかと言えばそうではなく、Gmail以外の受信メールサーバで余計な悪影響が出る（メールが弾かれる）可能性があります。なのでDMARC導入のベストプラクティス通り、ポリシー`p=none`から始めるのが無難と思います。まずは`p=none`にして、ruaレポートを受け取りながらSPF/DKIM認証の状況を確認しましょう。
`p=none`は既存のメール受信判定に影響を与えるべきではない[^5]とRFCに定められているので、設定して損はないはずです。自社から送っている正当なメールの中にGmailの要件を満たさないものがないか確認するために、できるだけ早く`p=none`にしてruaレポートを受け取ることをおすすめします。

なお`p=none`がゴールではないことは忘れないでください。Gmailの要件は`p=none`でOKですが、`p=none`は自社の正当なメールがSPF/DKIMに失敗していないかを確かめる過渡期のポリシーです。なりすましメールを真の意味で防止するには、レポートを受け取りながらしかるべき設定を行い、`p=quarantine`や`p=reject`に変更していく必要があります。
レポートはXML形式で書かれており見にくいので、解析のためのサービス[^6]を利用するのが良いです。

[^5]: [RFC7489 6.7. Policy Enforcement Considerations](https://datatracker.ietf.org/doc/html/rfc7489#section-6.7)の後ろから二段落目「To enable Domain Owners...」あたり参照

[^6]: [Cloudflare](https://blog.cloudflare.com/dmarc-management/), [DMARC/25](https://www.dmarc25.jp/), [Dmarcian](https://dmarcian.com/dmarc-saas-platform/), [PowerDMARC](https://powerdmarc.com/ja/), [Valimail](https://www.valimail.com/)等

### Reverse DNS【★★★】
要件箇条書きの二番目です。
> Ensure that sending domains or IPs have valid forward and reverse DNS records, also referred to as PTR records.

メール送信元IPアドレスの逆引き（Reverse DNS）が設定されていることを要求しています。Reverse DNSとはIPアドレスからドメイン名を取得することで、ドメインからIPアドレスを取得する正引きとの対比で「逆引き」と表現されます。
送信元IPアドレスからドメイン名を逆引きできるだけではなく、そのドメインのAレコードがちゃんと送信元IPアドレスになっていることも重要です。IPアドレスとドメインを互いに参照するプロセスのことをParanoid checkと言ったりForward-confirmed reverse DNS (FCrDNS)と言ったりします。

#### 実例
手元のメールで確認してみましょう。Gmailで「メッセージのソースを表示」します。「SPF: PASS」の表示があればそこに送信元IPアドレスが表示されています。または、メールソースの中のReceivedヘッダにも送信元IPアドレスが記載されます。

下の図はAmazon SESを利用しているconnpassさんのメールから拝借しました。送信元IPアドレスは`54.240.11.5`です。
![](https://storage.googleapis.com/zenn-user-upload/da49e14d0592-20231120.png)

Googleの[Digツール](https://toolbox.googleapps.com/apps/dig/#PTR/)のPTRレコードで`54.240.11.5`と入力すると`a11-5.smtp-out.amazonses.com`というドメインが返ってきます。これが逆引きDNSです。このドメインはReceivedヘッダにも書かれていますね。さらに、このドメインの[Aレコードを調べる](https://toolbox.googleapps.com/apps/dig/#A/)と`54.240.11.5`になります。これがFCrDNSです。

:::message
送信元IPアドレスの逆引きドメインを、送信元Fromのドメインに一致させる必要まではないだろうと考えられます。メール配信サービス等で他のユーザとIPアドレスを共有している場合、逆引きドメインを自社ドメインにすることはできないため、対応不可能な企業も多いでしょう。Gmailの要件でもその一致までは明言されていません。
:::

#### メール配信サービスを使う場合
SendGridやAmazon SESを使って送信すると、最初から`sendgrid.net`や`amazonses.com`のサブドメインで逆引きできるようです。
また専用のIPアドレスを利用している場合、逆引きドメインを自社ドメインにする設定があるはずです。その場合は、逆引きドメインを自社ドメインに一致させることを検討しても良いでしょう。例えば`github.com`から来るSendGrid経由のメールを確認したところ、送信元IPアドレスは`192.254.113.10`で、whoisはSendGrid、逆引きは`o5.sgmail.github.com`でした。
https://sendgrid.kke.co.jp/blog/?p=13266

### 送信時のTLS接続【★★】
12月6日に新たに追加された要件で、12月7日現在、英語版にしか[記載](https://support.google.com/mail/answer/81126?hl=en#zippy=%2Crequirements-for-all-senders:~:text=Use%20a%20TLS%20connection%20for%20transmitting%20email.)がありません。

> Use a TLS connection for transmitting email. For steps to set up TLS in Google Workspace, visit Require a secure connection for email.

全ての送信者はGmailアカウントへの送信時にTLS接続が求められます。上の英文の二文目にGoogle Workspaceから送る設定方法のリンクが書いてありますが、Google Workspaceを使って送らない場合もTLS接続を使う必要があります。ここの文の繋がりは謎です。

自社のメールをGmailで受け取ってみて、下の図のように「To」の左に赤い鍵マークが表示されている場合、TLSで暗号化されていません（鍵アイコンの意味については[公式記事](https://support.google.com/mail/answer/6330403)を参照のこと）。また「▼」をクリックすると「セキュリティ: このメールはexample.jpで暗号化されませんでした」と表示されます。
![](https://storage.googleapis.com/zenn-user-upload/5f0c99b86460-20231207.png)

メール配信サービスを使っていればまず大丈夫なはずです。自社のオンプレ等でTLS接続なしで送っている場合はメールサーバの設定を見直す必要があります。

### 迷惑メール率【★★★★】
発表当初の記載は
> Keep spam rates reported in Postmaster Tools below 0.3%.

だったのですが、12月7日現在は以下に変わっています。
> Keep spam rates reported in Postmaster Tools below 0.10% and avoid ever reaching a spam rate of 0.30% or higher.

「[迷惑メール率](https://support.google.com/mail/answer/9981691?hl=ja#zippy=%2C%E8%BF%B7%E6%83%91%E3%83%A1%E3%83%BC%E3%83%AB%E7%8E%87
)」とは、受信トレイに配信されたメールのうち、受信者が手動で迷惑メールに分類した件数の割合です。Gmailアカウントに5,000通送っているなら、迷惑メール報告数は5未満に抑えないといけません。0.1%と0.3%にどういう区別があるのかわかりにくいですが、定常的に0.1%を超えないよう運用して、突発的に高くなっても0.3%以上にならないようにしてね、という感じでしょうか。[FAQ](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cwhat-happens-when-sender-spam-rate-exceeds-the-maximum-spam-rate-allowed-by-the-guidelines:~:text=Currently%2C%20spam%20rates%20greater%20than%200.1%25%20have%20a%20negative%20impact%20on%20email%20inbox%20delivery.)によれば、0.1%を超えていると現状でも受信トレイに届きにくいようです。

現状どれだけ迷惑メール報告があるか知るには[Postmaster Tools](https://support.google.com/mail/answer/9981691)を設定する必要があります。ここで、Googleアカウントに送信したメールの迷惑メール率を確認できます。

このPostmaster Toolsですが、新要件を満たしているか確認できる機能が2024年初頭に追加されるそうなので、設定しない手はないです。
> In early 2024, Google plans to add a compliance status dashboard to Postmaster Tools. 

https://support.google.com/mail/answer/14289100?hl=en

#### メール配信サービスを使う場合の注意点
迷惑メール率についてはメール配信サービスにも似た機能があって、SendGridであれば[Spam Report](https://sendgrid.kke.co.jp/docs/User_Manual_JP/Suppressions/spam_reports.html)、Amazon SESであれば[Complaint](https://docs.aws.amazon.com/ja_jp/pinpoint/latest/userguide/channels-email-deliverability-dashboard-bounce-complaint.html)という形で迷惑メール報告を確認できます。しかしこれには**Googleアカウントの迷惑メール報告は含まれない**ので注意が必要です。メール配信サービス側が迷惑メール報告を検知するためには、メールボックスプロバイダ側が迷惑メール報告があったことを知らせないといけません。この仕組みをフィードバックループと言い、フィードバックループに対応したメールボックスプロバイダの迷惑メール報告のみ、メール配信サービス側で確認できます。そして、Gmailはこの形式でのフィードバックループに対応していません。[SendGrid](https://docs.sendgrid.com/ui/sending-email/google-feedback-loop)、[Amazon SES](https://docs.aws.amazon.com/ja_jp/ses/latest/dg/sending-email-suppression-list.html#:~:text=Gmail%20%E3%81%A7%E3%81%AF%E3%80%81SES%20%E3%81%AB%E8%8B%A6%E6%83%85%E3%83%87%E3%83%BC%E3%82%BF%E3%81%8C%E6%8F%90%E4%BE%9B%E3%81%95%E3%82%8C%E3%81%BE%E3%81%9B%E3%82%93%E3%80%82)それぞれのドキュメントにもそのことが記載されています。

SendGridやSESを使っているなら、ひとまず「Spam Report」や「Complaint」を見てみるのは構わないでしょう。ただし上述の通りフィードバックループに対応している受信サーバのものしか含まれないので、過小評価されている可能性があります。Gmailの迷惑メール報告を確認するには、Postmaster Toolsを設定して、そこで確認するしかありません。

#### 改善方法
もし迷惑メール率が高いことがわかったら、以下のような対策が必要です。
##### 簡単に配信停止できるようにする
配信停止プロセスが複雑だと、受信者は配信停止の代わりに迷惑メール報告してしまいます。メール内の配信停止リンクや、今回の要件に含まれる「one-click unsubscribe」を配置しましょう。

##### 受信を望む宛先に送信する
購入したリストに送信したり、オプトインしていない宛先に送信したりして、来ることを期待していないメールが届くと迷惑メール報告されやすいです。[オプトイン](https://sendgrid.kke.co.jp/blog/?p=4619)はきちんと取りましょう。

##### DMARCポリシーを厳しくする
自社を騙るなりすましメールが送信されている場合、DMARCポリシーを`p=quarantine`や`p=reject`にすることで、受信者が迷惑メール報告する前に弾くことができます。

### RFC5322の準拠【★★】
> Format messages according to the Internet Message Format standard (RFC 5322).

RFC5322ではメールのメッセージの形式が定義されています。メールソースの各行は998文字を超えてはいけませんとか、Toヘッダが複数回現れてはいけませんとか、そういうことが書いてあります。形式違反は現状でも届かないことがありますが[^7]、今後はさらに厳しくなる可能性があります。

メール配信サービスを使っている場合、RFC5322に準拠するように送ってくれるはずなので、あまり気にしなくて良い要件です。

> クライアントが Amazon SES にリクエストを送ると、Amazon SES では、Internet Message Format 仕様（RFC 5322）に準拠した E メールメッセージを構築します。

https://docs.aws.amazon.com/ja_jp/ses/latest/dg/send-email-concepts-email-format.html

[^7]: 例えばヘッダ重複は[バウンスします](https://support.google.com/a/answer/13567860?hl=ja)。

### 送信元Fromドメインに`gmail.com`を使用しない【★★】
個人のGmailアカウントから送るわけではないのに送信元Fromに`gmail.com`を使ったらなりすましです。当然避けるべきです。
技術的には、期日以降`gmail.com`ドメインのDMARCポリシーに`p=qurantine`が設定されるようです。したがって、こういうメールは迷惑メールボックス行きになります。
きちんと自分のドメインを使ってメール送信しているなら気にしなくて良い要件です。

### メール再配送時のARCヘッダ付与【★】
メールの転送やメーリングリストなど、一度受け取ったメールをGoogleアカウントに再配送するような送り方をしている送信者が対象です。メールの転送やメーリングリストは最終的な宛先でSPF/DKIM認証が失敗することが多いので、再配送するたびごとに認証結果をメールヘッダに記載して、認証のチェーンを作る仕組みがARCです。

再配送するような送り方をしていなければ（単に企業から顧客へ直接メールを届けるようなケースでは）対応の必要はないはずです。
https://www.naritai.jp/technology_arc.html

### 配信停止【★★★】
受信者が簡単に配信停止できるようにすることは開封率、クリック率、迷惑メール率、到達率全てに良い影響を与えます。そのための要件です。

> Marketing messages and subscribed messages must support one-click unsubscribe, and include a clearly visible unsubscribe link in the message body.

Gmailのガイドラインの[日本語訳](https://support.google.com/mail/answer/81126?hl=ja#requirements-5k&zippy=%2C%E6%97%A5%E3%81%82%E3%81%9F%E3%82%8A-%E4%BB%B6%E4%BB%A5%E4%B8%8A%E3%81%AE%E3%83%A1%E3%83%BC%E3%83%AB%E3%82%92%E9%80%81%E4%BF%A1%E3%81%99%E3%82%8B%E5%A0%B4%E5%90%88%E3%81%AE%E8%A6%81%E4%BB%B6)を読むと意図を取り違えてしまうかもしれないのですが、英語で読むと条件は以下の2つあることがわかります。

- 「one-click unsubscribe」に対応すること
- メッセージ本文に登録解除のリンクをわかりやすく表示すること

これ、本文内にリンクを入れるだけではなく、メールヘッダにもきちんと配信停止の仕組みを取り入れなさいよと言っています。それが一つ目の「one-click unsubscribe」です。メールヘッダにしかるべき情報を入れておくことで、ワンクリックで配信停止できるボタンをメールボックスプロバイダ側が用意してくれます。

:::message
メール本文に配信停止リンクを入れさえすればOKなのか、メールヘッダにも配信停止の仕組みを入れる必要があるのか、は解釈が分かれるかもしれません。そういう質問があったのか、FAQにはヘッダのことだと[明記](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined%2Cdo-all-messages-require-one-click-unsubscribe%2Cif-the-list-header-is-missing-is-the-message-body-checked-for-a-one-click-unsubscribe-link%2Cif-unsubscribe-links-are-temporarily-unavailable-due-to-maintenance-or-other-reasons-are-messages-flagged-as-spam%2Ccan-a-one-click-unsubscribe-link-to-a-landing-or-preferences-page:~:text=No.%20One%2Dclick%20unsubscribe%20should%20be%20implemented%20according%20to%20RFC%208058)されています。またGmailの新要件アナウンスのブログにも登場している、Yahoo!のMarcel Becker (Sr. Director Product Management) が出た[ウェビナー](https://www.validity.com/resource-center/state-of-email-live-gmail-yahoos-new-sender-requirements/)を見たところ、この要件はヘッダに対するものであると再三述べていました。
> When we talk about "one-click unsubscribe" requirements, this is for the HEADERS, in the list-unsubscribe header. It's not about a link you have in the body.
:::

#### 表示の例
「ワンクリックで配信停止できるボタン」がどう見えるのか確認しておきましょう。以下にGmailとYahoo!の例を示します。どちらも、Fromアドレスの横という目立つ場所にメール登録解除のリンクが表示されています（図に示したように他の場所にもリンクはあります）。これをクリックすると右側のモーダルが表示され、青いボタンをクリックすると配信停止されます。

![](https://storage.googleapis.com/zenn-user-upload/4dbc67b08bd8-20231128.png)
*Gmailの場合*

![](https://storage.googleapis.com/zenn-user-upload/747e4e332083-20231128.png)
*米国Yahoo!の場合*

このような表示は特定の条件のもとでメールクライアントが勝手に表示してくれるものです。今回の要件である「ワンクリックで配信停止できること」は、この表示が出るようにすること、と読み替えてもいいかもしれません。自分が送っているメールにこの表示があるかどうか確認しておきましょう。なかったら対応が必要かもしれません。

:::message alert
ただしGmailにおいてはこの表示が出る条件は単純ではなさそうです。手元で試すと、メールヘッダや本文が同じような内容でも、送信元From（ヘッダFrom）が異なると出たり出なかったりしました。したがって、表示条件はヘッダがあるかどうかだけではなく、大量送信者かどうか等も見ている気がします（対応できてるか分かりづらいのでやめてほしい）。この辺りの最終的な判断基準はGoogleに聞いてみないとわからなそうです[^8]。
:::

[^8]: とはいえどうやって聞けば良いんだと思ってましたが、FAQに[問い合わせ先が記載](https://support.google.com/a/answer/14229414?hl=en#:~:text=Support%20%26%20escalation)されていました。

#### 仕様の解説
「one-click unsubscribe」は[RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058)で定義されている用語で、HTTPのPOSTリクエストで配信停止を行うものです。なのでRFC 8058に準拠するのが理想だとは思いますが、Yahoo!の要件ではMAILTOによる配信停止（[RFC 2369](https://datatracker.ietf.org/doc/html/rfc2369)）でもOKだと言っています[^9]。実際Gmailのワンクリック配信停止リンクも、HTTPでなくても表示されるようです。
それぞれ説明します。

[^9]: 要件を記した[このページ](https://senders.yahooinc.com/subhub)に「Of course we support “mailto:” unsubscribe headers as well.」と書いてある。

##### `HTTP`による方法（RFC 8058）
これはGmailのガイドラインに[記載](https://support.google.com/mail/answer/81126?hl=en#subscriptions)されている通りですが、以下の2つのヘッダをメールに追加する必要があります。
- `List-Unsubscribe: <https://example.jp/unsubscribe>`
- `List-Unsubscribe-Post: List-Unsubscribe=One-Click`

メールクライアントが表示する配信停止のリンクをクリックすると、`https://example.jp/unsubscribe`に対してPOSTリクエストが送られます。ここでは例として単純なURLを書いていますが、実際には宛先ごとに異なるURLを用意し、どの宛先がクリックしたか判別できるようにしておきます。POSTリクエストを受け取ったら、その宛先には配信を行わないようにします。

:::message alert
メール本文に配置した配信停止リンクの飛び先として、Preference center（どんな種類のメールを受け取るか選ぶ画面）や、サービスのログイン画面を指定している場合もあると思います。しかし、単にそのURLを`List-Unsubscribe`に指定するのはだめです。これらのページは遷移後に受信者の操作を必要としますが、それは「one-click」ではありません。あくまでメールクライアント内のボタンを「one-click」するだけで配信停止できるようにする必要があります。RFC 8058はそのことを規定した仕様です。GmailのFAQにも[記載あり](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined%2Cdo-all-messages-require-one-click-unsubscribe%2Cif-the-list-header-is-missing-is-the-message-body-checked-for-a-one-click-unsubscribe-link%2Cif-unsubscribe-links-are-temporarily-unavailable-due-to-maintenance-or-other-reasons-are-messages-flagged-as-spam%2Ccan-a-one-click-unsubscribe-link-to-a-landing-or-preferences-page:~:text=Can%20a%20one%2Dclick%20unsubscribe%20link%20to%20a%20landing%20or%20preferences%20page%3F)。
:::

##### `MAILTO`による方法（RFC 2369）
`List-Unsubscribe`ヘッダは元々RFC 2369で定義されており、ここでは`MAILTO`を使った配信停止方法が例示されています。
- `List-Unsubscribe: <mailto:unsubscribe@example.jp?subject=unsubscribed-user-identifier>`

メールクライアントが表示する配信停止のリンクをクリックすると、`unsubscribe@example.jp`にメールが送られます。上の例ではSubjectに`unsubscribed-user-identifier`という文字列が入ることになります。Subjectではなくメールアドレスのローカルパートとかでも良いですが、どの宛先がクリックしたか見分けがつくようにします。
`MAILTO`による方法では`List-Unsubscribe-Post`ヘッダは必要ありません。

#### 対応方法
どのように配信停止を管理しているかによってやり方や難易度が異なると思われます。
##### メール配信サービスで配信停止を管理している場合
SendGridやAmazon SES等では配信停止の仕組みを提供していて、本文内へのリンクやList-Unsubscribeヘッダの挿入、配信停止した宛先リストの管理までできます。メール配信サービスに配信停止を全て任せているという場合は、追加の対応は必要ないと思われます。

https://docs.aws.amazon.com/ja_jp/ses/latest/dg/sending-email-subscription-management.html

純粋なメール配信サービスだけではなく、メール配信機能を持つ顧客管理（CRM）ツールやマーケティングオートメーション（MA）ツールでも配信停止の仕組みを提供しているものがあります。その場合も、メール配信サービスと同様に利用者側の対応は必要ないと思われます。

https://help.salesforce.com/s/articleView?id=000386352&type=1

顧客管理はCRMやMAでやっているがメール送信自体はSendGridのアカウントを作ってAPIキーを連携しているみたいなパターンは、配信停止の仕組み自体はCRM/MA側が持っていると思います。状況はCRM/MA側に問い合わせてみるのが良いと思います。

##### 自社で配信停止を管理している場合
自社のサイトにPreference center（どんな種類のメールを受け取るか選ぶ画面）を用意していたり、ログイン後のページで配信停止を選ばせたりしているケースです。この場合は少し大変で、ユーザの操作なしに配信停止を受け付けるための一意のURLやメールアドレスを用意し、`List-Unsubscribe`ヘッダに`http`や`mailto`として指定する必要があります。

#### 対応期限
GmailのFAQに気になる[項目](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined%2Cdo-all-messages-require-one-click-unsubscribe%2Cif-the-list-header-is-missing-is-the-message-body-checked-for-a-one-click-unsubscribe-link%2Cif-unsubscribe-links-are-temporarily-unavailable-due-to-maintenance-or-other-reasons-are-messages-flagged-as-spam%2Ccan-a-one-click-unsubscribe-link-to-a-landing-or-preferences-page:~:text=Do%20all%20messages%20require%20one%2Dclick%20unsubscribe%3F)があります。

> No. One-click unsubscribe is required only for commercial, promotional messages. Transactional messages are excluded from this requirement. Some examples of transactional messages are password reset messages, reservation confirmations, and form submission confirmations.
> 
> Senders that already include an unsubscribe link in their messages have until June 1st 2024 to implement one-click unsubscribe in all commercial messages.

二段落目を訳すと「本文内に配信停止リンクを入れている場合は2024年6月1日までにone-click unsubscribeに対応してください」となります。つまり、one-click unsubscribeの対応だけは猶予が設けられているということなんでしょうかね。実装が大変だからでしょうか。真相はよくわかりません。

## おわりに
2023年12月7日時点でのGmailの新要件とその対応方法を全てまとめてみました。

今回の新要件の重要な要素は「送信ドメイン認証をしっかり通すこと」「迷惑メール率を低く抑えること」「配信停止を簡単にできるようにすること」の3つです。これらは正しいメールをきちんと届けるためのベストプラクティスとして昔から言われていることであり、今回の新要件はそれを明確化したものです。いささか期限が短いですが、今回の新要件はメール配信の質を向上させるためのものなので、できるだけ早く対応する必要がありそうです。

## よくある質問
### この要件は、送信側がGoogleアカウント（Google Workspace等）を使っている場合のみ適用されますか？
いいえ。送信側がGoogleアカウントを使っているかどうかは関係ないでしょう。送信先がGmailアカウントであれば、どこから送っていようがこの要件が適用されると思われます。

### SendGridやAmazon SESなどのメール配信サービスを使っていれば、万事OKですか？
いいえ。メール配信サービスはDMARCまではサポートしてくれないです。SPFとDKIMは送るメールそれぞれに対する認証ですが、DMARCはドメイン（組織）としてのポリシーの表明です。SPFやDKIMは送信メールサーバでしか知り得ない情報（送信元IPアドレスやメールヘッダの電子署名）を使うのでサービス側にそれ用の設定がありますが、DMARCポリシーは自分で決めて自分で公開しなければなりません。
また、迷惑メール率もメール配信サービスが保証してくれるものではありません。メールのコンテンツは送信者が作成するものなので、迷惑メール報告も送信者自身の責任です。

### 5,000通/日送信しているかわかりません。どうすればいいですか？
迷う場合は5,000通以上送信していると思って厳しい方の条件を満たしましょう。いずれ、すべての送信者がこれらの条件を満たすことが求められるようになるんじゃないでしょうか。

### 全てのメールに配信停止リンクを入れる必要がありますか？
いいえ。マーケティングメールが対象です。パスワードリセットや購入確認等のいわゆる「トランザクションメール」は対象外です。GmailのFAQに[記載](https://support.google.com/a/answer/14229414?hl=en#zippy=%2Cif-messages-fail-dmarc-authentication-can-they-be-delivered-using-ip-allow-lists-or-spam-bypass-lists-or-will-these-messages-be-quarantined%2Cdo-all-messages-require-one-click-unsubscribe%2Cif-the-list-header-is-missing-is-the-message-body-checked-for-a-one-click-unsubscribe-link%2Cif-unsubscribe-links-are-temporarily-unavailable-due-to-maintenance-or-other-reasons-are-messages-flagged-as-spam%2Ccan-a-one-click-unsubscribe-link-to-a-landing-or-preferences-page%2Cwhat-is-a-bulk-sender:~:text=Do%20all%20messages%20require%20one%2Dclick%20unsubscribe%3F)があります。

### 米国Yahoo!の要件は、日本のYahoo! JAPANにも適用されますか？
Yahoo! JAPANのメールサービスは米国Yahoo!とは独立しており、今回の発表とは無関係と考えられます。もはや米国Yahoo!のグループ会社でもありませんし、「[ブランドアイコン](https://announcemail.yahoo.co.jp/brandicon_corp/)」表示など独自のスパムメール対策を行っています。ブログ等の情報発信も多いので、もし今回の規制強化に追随する場合も何らかの発表があることを期待しています。

## 参考リンク集
最後に、筆者が参考にした資料をまとめておきます。
### Google
アナウンスのブログ
https://blog.google/products/gmail/gmail-security-authentication-spam-protection/
ガイドライン（日本語版）
https://support.google.com/mail/answer/81126?hl=ja
ガイドライン（英語版）
https://support.google.com/mail/answer/81126?hl=en
FAQ（英語版）
https://support.google.com/a/answer/14229414?hl=en
FAQは随時更新されるそうなので定期的に見た方が良さそうです。まだ日本語版はありません。
ガイドラインも英語版が最新かつ正しいと思われます。

### Yahoo!
アナウンス
https://blog.postmaster.yahooinc.com/post/730172167494483968/more-secure-less-spam
https://senders.yahooinc.com/subhub/

### M3AAWG
M3AAWGはメッセージングやマルウェア等の不正利用の対策を協議する国際的な非営利組織です。Google、米Yahoo!、Microsoft、AWS、SendGridなどがメンバーです。
ここの情報も一次資料と思って良いでしょう。
https://www.m3aawg.org/blog/SendingBulkMailToGmail_Yahoo
### その他
当事者以外の企業や個人の情報発信も多く、これらも参考になります。ただし最新の情報が反映されていない可能性があるので注意が必要です。
https://www.proofpoint.com/jp/blog/email-and-cloud-threats/google-and-yahoo-set-new-email-authentication-requirements
https://sendgrid.kke.co.jp/blog/?p=17031
https://www.m3tech.blog/entry/2023/10/24/110000
https://happy-nap.hatenablog.com/entry/2023/12/01/224252
https://maildata.jp/specification/google.html
https://qiita.com/nfujita55a/items/37b05801209f6058808e