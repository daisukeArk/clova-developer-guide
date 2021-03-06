## HTTPメッセージ {#HTTPMessage}
CEKとExtensionが通信する際、HTTP/1.1を使用して基本的なHTTPSリクエストとHTTPSレスポンスをやり取りします。CEKとExtensionが通信する際、HTTPメッセージのボディには、JSON形式のメッセージが含まれます。ここでは、CEKとExtensionがやり取りするHTTPメッセージの構成について説明します。

* [HTTPヘッダー](#HTTPHeader)
* [HTTPボディ](#HTTPBody)
* [リクエストメッセージを検証する](#RequestMessageValidation)

### HTTPヘッダー {#HTTPHeader}
CEKがExtensionに解析されたユーザーの発話情報を渡す際、HTTPリクエストを使用します。その際、HTTPリクエストのヘッダーは、次のように構成されます。

{% raw %}

```
POST /APIpath HTTP/1.1
Host: YOUR_EXTENSION_ENDPOINT
Content-Type: application/json;charset-UTF-8
Accept: application/json
Accept-Charset: utf-8
SignatureCEK：{{ SignatureCEK }}
```
{% endraw %}

* HTTP/1.1バージョンでHTTP通信し、POSTメソッドを使用します。
* Hostとリクエストパスは、Extensionの開発者があらかじめ定義したURIに設定されます。
* リクエストボディのデータはJSON形式で、UTF-8エンコーディングを使用します。
* `SignatureCEK`フィールドとRSA公開鍵を使用して、[Clovaから送信されたリクエストかどうかを検証](#RequestMessageValidation)することができます。

逆に、ExtensionがClovaに処理結果を返す際、HTTPレスポンスを使用します。その際のHTTPレスポンスヘッダーは、以下のように設定をしてください。
{% raw %}
```
HTTP/1.1 200 OK
Content-Type: application/json;charset-UTF-8
```
{% endraw %}
* CEKから渡されたHTTPリクエストに対するレスポンスとして、処理結果を返します。
* ボディのデータはJSON形式で、UTF-8エンコーディングを使用します。

### HTTPボディ {#HTTPBody}
リクエストメッセージとレスポンスメッセージのボディはJSON形式で、解析されたユーザーの発話情報や、Extensionの処理結果が含まれています。それぞれのメッセージの構成は、使用するExtensionの種類によって異なります。メッセージ構成の詳細については、[Custom Extensionメッセージ](#CustomExtMessage)を参照してください。

### リクエストメッセージを検証する {#RequestMessageValidation}
ExtensionがCEKからHTTPリクエストを受信するとき、そのリクエストが第三者ではなく、Clovaから送信された信頼できるリクエストかどうかを検証する必要があります。[HTTPヘッダー](#HTTPHeader)にある`SignatureCEK`フィールドとRSA公開鍵を使用して、以下のようにリクエストメッセージを検証してください。

**RSA公開鍵を用いてリクエストメッセージを検証する**
<ol>
<li><p>`context.System.application.applicationId`が設定済みの`ExtensionId`と同一であることを確認してください</p></li>
<li><p>Clovaの署名用RSA公開鍵を以下のURLからダウンロードしてください</p>
<p>https://clova-cek-requests.line.me/.well-known/signature-public-key.pem</p></li>
<li><p><code>SignatureCEK</code>ヘッダーの値を取得してください</p>
<p><code>SignatureCEK</code>ヘッダーの値は、Base64エンコードされた、HTTP bodyの<a href="https://tools.ietf.org/html/rfc3447" target="_blank">RSA PKCS #1 v1.5</a>署名値です。</p></li>
<li>ステップ1でダウンロードしたRSA公開鍵を用いてステップ2で取得した<code>SignatureCEK</code> ヘッダーを以下のように<a href="https://tools.ietf.org/html/rfc3447#section-5.2" target="_blank">検証(verify)</a>してください</li>
</ol>

```java
String signatureStr = req.getHeader("SignatureCEK");
byte[] body = getBody(req);
String publicKeyStr = downloadPublicKey();
publicKeyStr = publicKeyStr.replaceAll("\\n", "")
    .replaceAll("-----BEGIN PUBLIC KEY-----", "")
    .replaceAll("-----END PUBLIC KEY-----", "");
X509EncodedKeySpec pubKeySpec = new X509EncodedKeySpec(Base64.getDecoder().decode(publicKeyStr));
KeyFactory keyFactory = KeyFactory.getInstance("RSA");
PublicKey pubKey = keyFactory.generatePublic(pubKeySpec);
Signature sig = Signature.getInstance("SHA256withRSA");
sig.initVerify(pubKey);
sig.update(body);
byte[] signature = Base64.getDecoder().decode(signatureStr);
boolean valid = sig.verify(signature);
```

<div class="note">
  <p><strong>メモ</strong></p>
  <p>検証に失敗した場合にはそのリクエストは破棄してください。</p>
</div>

<div class="danger">
  <p><strong>注意</strong></p>
  <p>HTTP headerのフィールドは大文字/小文字を区別しないcase-insensitiveです。<a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.2" target="_blank">Section 4.2, "Message Headers"</a></p>
  <p>実装時にはフィールド名の大文字/小文字問わず、値が取得できるようにしてください。</p>
</div>

