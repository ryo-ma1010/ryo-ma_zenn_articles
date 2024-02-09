---
title: "reCAPTCHA v2をEC-CUBE2系に実装する"
emoji: "🐙"
type: "tech"
topics:
  - "eccube"
  - "recaptcha"
published: true
published_at: "2022-08-08 16:02"
---

:::message alert
この記事は自分が社内でまとめていたものを転載しております。
情報が古い恐れがあります。ご了承ください。
投稿日：2022/01/18
:::

# 環境
- EC-CUBE2系
- reCAPTCHA v2
- PHP 5.4

# 背景
ある案件でreCAPTCHA v2をeccubeに設置することになりました。

# reCAPTCHA
googleが提供する「私はロボットではありません」チェックボックスです。
今回はv2を利用します。

https://developers.google.com/recaptcha/docs/display

# 大まかな流れ
* google reCAPTCHA管理画面からreCAPTCHAを使えるようにする
* ECCUBEへの実装
* 実際に動くか検証

# 詳細
## google reCAPTCHA管理画面からreCAPTCHAを使えるようにする

管理画面から下記のようなことを書いてreCAPTCHAを使えるようにする。
ドメインを複数登録でき、localhostも登録できるものの、グローバルな名前なので非推奨です。

![](https://storage.googleapis.com/zenn-user-upload/39202f87fcb0-20220808.png)

 **登録後、サイトキーとシークレットキーを入手することができるのですが、実装時に必要になりますので、大切に保管してください。** 


## ECCUBEへの実装

今回は、会員登録ページ（入力ページ）と購入確認ページに実装しました。

### config.php
実装にはサイトキーとシークレットキーが必要です。
各種キーはgit管理外で保存したいため、config.phpに登録しました。

#### data/config/config.php

```php
// reCAPTCHA sitekey
define('G_RECAPTCHA_SITEKEY', 'hogehogehoge');
// reCAPTCHA secretkey
define('G_RECAPTCHA_SECRETKEY', 'hugahugahuga');
```

***

### テンプレートファイル
下記ページを参照して、実装したいページのテンプレートファイルにタグを設置します。

https://developers.google.com/recaptcha/docs/display#auto_render

#### data/Smarty/templates/default/entry/index.tpl

```html
<!--{if $isReCAPTCHA}-->
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
<!--{/if}-->

～中略～

<!--{if $isReCAPTCHA}-->
<!--{if $arrErr.g_recaptcha}-->
        <div class="attention"><!--{$arrErr.g_recaptcha}--></div>
<!--{/if}-->
<div class="g-recaptcha" data-sitekey="<!--{$smarty.const.G_RECAPTCHA_SITEKEY}-->"></div>
<!--{/if}-->
```

***

### PHPファイル
「私はロボットではありません」にチェックをし、送信ボタンを押した後のサーバサイドの処理を記述します。
サーバにPOSTでトークンが送られてくるので、シークレットキーと合わせてgoogleが指定するAPIにcurlします。
今回は入力項目のエラーチェックと並行して実装しようとしたので以下のようなメソッドを作りました。
APIの結果がtrueの場合はNULLでfalseの場合は何かしらのエラーメッセージを返します。

#### data/class_extends/page_extends/entry/LC_Page_Entry_Ex.php

```php
// reCAPTCHAタグ
$reCAPTCHAResponse = $this->siteVerifyRecaptcha($_POST['g-recaptcha-response']);
if ($reCAPTCHAResponse != "") {
    $this->arrErr['g_recaptcha'] = $reCAPTCHAResponse;
}

```

***

#### data/class_extends/page_extends/LC_Page_Ex.php

```php
public function isReCAPTCHA()
{
    $this->isReCAPTCHA = G_RECAPTCHA_SITEKEY != "" && G_RECAPTCHA_SECRETKEY != "";
}

/**
 * reCAPTCHAチェックするメソッド
 */
public function siteVerifyRecaptcha($gRecaptchaResponse)
{
    if ($this->isReCAPTCHA === false) {
         return;
    }

    if ($gRecaptchaResponse == "") {
        return "※ reCAPTCHAが選択されていません。<br />";
    }

    $ch = curl_init();
    $postFields = array(
        'secret' => G_RECAPTCHA_SECRETKEY,
        'response' => $gRecaptchaResponse,
        'remoteip' => $_SERVER['REMOTE_ADDR'],
    );
    $option = array(
        CURLOPT_URL => 'https://www.google.com/recaptcha/api/siteverify',
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POSTFIELDS => http_build_query($postFields),
    );
    curl_setopt_array($ch, $option);
    $jsonResponse = curl_exec($ch);
    curl_close($ch);

    $response = json_decode($jsonResponse, true);
    
    if ($response['success']) {
        GC_Utils_Ex::gfPrintLog('reCAPTCHA : success.', '', false);
        return;
    } else {
        GC_Utils_Ex::gfPrintLog('reCAPTCHA : error.' . print_r($response['error-codes'], true), '', false);
        return "※ reCAPTCHAでエラーが発生しました。お手数ですが、もう一度やり直してください。<br />";
    }
}
```

## 検証

実際に動くかどうか確認します。
※チェックボックスを押したとき、2分以内に送信ボタンを押さないとタイムアウトしてしまうのですが、タイムアウト時間は変更できません。

![](https://storage.googleapis.com/zenn-user-upload/62c38ee50aaa-20220808.png)

### data/logs/site.log

```
2022/01/18 18:50:01 [/entry/index.php] reCAPTCHA : success. from ip.hoge.huga.piyo
```

## まとめ
v2は比較的簡単に実装することができました。
実装後のスタイル調整等にもよると思いますが1人日あれば実装できるかなと思います。