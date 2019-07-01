# Bodygram SDK JS Readme
Bodygramは、画像を含むいくつかのパラメータを送信するだけで人体のサイズ計測を行えるシステムです。

このSDKで身体サイズの推定リクエストをBodygramサーバーに送信するだけで、20 ~ 40秒後に推定結果を受け取ることができます。

> Note: 本SDKはフロントエンドでの利用を想定しています。

> Note: 本SDKはES6が動作するモダンなモバイル端末での利用を想定しています。

本SDKでは下記の機能をカバーしています。

1. トークンの作成・復元
2. 推定リクエストの送信
3. 推定結果の取得

> Note: Bodygramを動作させるためのUIについては、本SDKには含まれません。

## バージョン
2019-06-30 SDK Version 0.0.1

## ブラウザ対応範囲
[browserlist](https://browserl.ist/?q=%3E1%25+and+not+ie+%3C%3D+11+and+not+op_mini+all)

## インストール
本リポジトリをクローンし、`bodybank-enterprise.min.js`をプロジェクト内に含めた上で、indexとなるhtmlファイルに下記のように読み込んで下さい。

```html
<head>
  <meta charset="UTF-8">
  <title>Your awesome website</title>
  <script type="text/javascript" src="./bodybank-enterprise.min.js"></script>
</head>
```

## 利用方法
TBD