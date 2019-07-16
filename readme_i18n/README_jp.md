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
### セットアップ
ご契約後にお渡しする`bodybank-config.json`をプロジェクト内に追加して下さい。

### Token Provider
Token Providerは、Bodygramのバックエンドリソースにアクセスするためのトークン発行・管理を担うクラスです。

- Default Token Provider
- Direct Token Provider

の二種類が存在し、前者はproduction向け、後者は開発環境向けの実装となります。

下記で各々の詳細を説明します。

#### Direct Token Provider
まずは開発環境向けのDirect Token Providerについてです。

本機能は開発環境向けであるため、プロダクションリリース時には必ず次項のDefault Token Providerにて実装をお願いします。

`DirectTokenProvider`は `DefaultTokenProvider`の子クラスであり、tokenProviderに必要な設定を簡略化できます。

```javascript
const bodybank = new BodyBankEnterprise()
const tokenProvider = bodybank.getDirectTokenProvider("https://api.<SHORT IDENTIFIER>.enterprise.bodybank.com", "YOUR API KEY")

tokenProvider.tokenDuration = 86400
tokenProvider.userId = "encryptedUserId"
```

以上でDirect Token Providerの設定は完了です。

#### Default Token Provider
プロダクション向けのDefault Token Providerについて解説します。

Default Token Providerを利用する場合、開発ステップは下記のようになります。

1. 貴社で用意したAPIサーバーに、トークン取得処理を行うエンドポイントを作成
2. Bodygramを利用するフロントエンドプロジェクトにて、上記エンドポイントをコールする

詳細を解説します。

仮に、貴社でお持ちのAPIサーバーが`https://awesome-api.com`にて動作しているとします。
Bodygramのトークン取得を行うエンドポイントは`https://awesome-api.com/bodygram/token`に対するGETリクエストで叩けるとします。

上記エンドポイントにルーティングされるファイルに下記のような実装を行います。
(仮にNode.js + Expressでの実装だとします)

```javascript
const rp = require('request-promise')

export default async function (req, res) {
  const apiKey = "YOUR API KEY"
  const apiEndpoint = "https://api.<SHORT IDENTIFIER>.enterprise.bodybank.com"
  const userId = 'encryptedUserId' // 簡略化のため固定値にしています
  const options = {
    headers: { "x-api-key": apiKey },
    json: { user_id: userId }
  }

  const tokenRes = await rp.post(apiEndpoint, options)
  
  res.send(tokenRes.content.token);
}
```

弊社からお渡しする`apiEndpoint`に対して`apiKey`及び、トークンが必要な`userId`を含めてPOSTします。

上記の準備ができたら、フロントエンドに戻り下記のような実装を行います。

```javascript
const bodybank = new BodyBankEnterprise()
const tokenProvider = bodybank.getDefaultTokenProvider()

tokenProvider.restoreTokenBlock = async () => {
  try {
    const tokenEndpoint = 'https://awesome-api.com/bodygram/token'
    const options = { method: "GET" }
    const res = await fetch(tokenEndpoint, options)
    const {
      content: {
        token: {
          jwt_token,
          identity_id
        }
      }
    } = await res.json()

    return bodybank.genBodyBankToken(jwt_token, identity_id)
  } catch (err) {
    const error = new Error(`Failed to fetch bodybank token.\nreason: ${err.message}`)

    throw error
  }
}
```

`restoreTokenBlock`は、トークンの期限が切れそうなとき、もしくは切れた時に自動的に呼ばれる関数です。

このように貴社独自のエンドポイントを作成して頂く理由は、apiEndpoint及びapiKeyが外部に露出しないことを保証するためです。

### SDKの初期化
リクエストの作成や推定結果の取得を行う前に、`BodyBankEnterprise`を初期化する必要があります。
初期化の際には上記で設定した`tokenProvider`及び`bodybankConfig`を渡します。

> Note: bodybankConfigは、`bodybank-config.json`をそのまま読み込んだものです

```javascript
bodybank.initialize(tokenProvider, bodybankConfig)
```

### webhookの準備
推定リクエストは非同期で実行されます。
推定リクエストのステータスを追うために、弊社ではwebhookの機能を提供しています。

webhookを利用するためには、事前に通知先となるAPIエンドポイント(POST)、及びAPIコールに必要なauthorization関連の情報などをご教示下さい。

> Note: 2019年6月時点で設定可能なAPIエンドポイントは1つのみです。

設定されたエンドポイントには、推定リクエストのステータスが変わるたびにその時点の推定リクエストの内容がPOSTリクエストで通知されます。

1. requested: 推定リクエストが作成された初期ステータス
2. pendingAutomaticEstimation: 推定待ちキューに入ったステータス
3. completed: 推定完了
4. failed: 推定失敗

下記に、各ステータスで通知されるrequest bodyの例を記載します。

Status: requested
```javascript
{
  notification_type: 'ESTIMATION_CREATED',
  request: {
    age: 45,
    bicep_circumference: null,
    calf_circumference: null,
    chest_circumference: null,
    created_at: 1561530296,
    updated_at: null,
    error_code: null,
    error_detail: null,
    front_image_url: null,
    front_thumbnail_image_url: null,
    gender: 'male',
    height: 173,
    high_hip_circumference: null,
    hip_circumference: null,
    id: 'uniqueRequestId',
    inseam_length: null,
    knee_circumference: null,
    neck_circumference: null,
    outseam_length: null,
    race: null,
    shoulder_width: null,
    side_image_url: null,
    side_thumbnail_image_url: null,
    sleeve_length: null,
    backlength: null,
    underbust: null,
    status: 'requested',
    thigh_circumference: null,
    mid_thigh_circumference: null,
    total_length: null,
    user_id: 'uniqueUserId',
    waist_circumference: null,
    weight: 68,
    wrist_circumference: null,
    fail_on_automatic_estimation_failure: true,
    probabilityOfBicepCircumferenceInHitZone: null,
    probabilityOfCalfCircumferenceInHitZone: null,
    probabilityOfChestCircumferenceInHitZone: null,
    probabilityOfHighHipCircumferenceInHitZone: null,
    probabilityOfHipCircumferenceInHitZone: null,
    probabilityOfInseamLengthInHitZone: null,
    probabilityOfKneeCircumferenceInHitZone: null,
    probabilityOfMidThighCircumferenceInHitZone: null,
    probabilityOfNeckCircumferenceInHitZone: null,
    probabilityOfOutseamLengthInHitZone: null,
    probabilityOfShoulderWidthInHitZone: null,
    probabilityOfSleeveLengthInHitZone: null,
    probabilityOfThighCircumferenceInHitZone: null,
    probabilityOfTotalLengthInHitZone: null,
    probabilityOfWaistCircumferenceInHitZone: null,
    probabilityOfWristCircumferenceInHitZone: null
  }
}
```

Status: pendingAutomaticEstimation
```javascript
{
  notification_type: 'ESTIMATION_STATUS_UPDATED',
  request: {
    age: 45,
    bicep_circumference: null,
    calf_circumference: null,
    chest_circumference: null,
    created_at: 1561530296,
    updated_at: null,
    error_code: null,
    error_detail: null,
    front_image_url: null,
    front_thumbnail_image_url: null,
    gender: 'male',
    height: 173,
    high_hip_circumference: null,
    hip_circumference: null,
    id: 'uniqueRequestId',
    inseam_length: null,
    knee_circumference: null,
    neck_circumference: null,
    outseam_length: null,
    race: null,
    shoulder_width: null,
    side_image_url: null,
    side_thumbnail_image_url: null,
    sleeve_length: null,
    backlength: null,
    underbust: null,
    status: 'pendingAutomaticEstimation',
    thigh_circumference: null,
    mid_thigh_circumference: null,
    total_length: null,
    user_id: 'uniqueUserId',
    waist_circumference: null,
    weight: 68,
    wrist_circumference: null,
    fail_on_automatic_estimation_failure: true,
    probabilityOfBicepCircumferenceInHitZone: null,
    probabilityOfCalfCircumferenceInHitZone: null,
    probabilityOfChestCircumferenceInHitZone: null,
    probabilityOfHighHipCircumferenceInHitZone: null,
    probabilityOfHipCircumferenceInHitZone: null,
    probabilityOfInseamLengthInHitZone: null,
    probabilityOfKneeCircumferenceInHitZone: null,
    probabilityOfMidThighCircumferenceInHitZone: null,
    probabilityOfNeckCircumferenceInHitZone: null,
    probabilityOfOutseamLengthInHitZone: null,
    probabilityOfShoulderWidthInHitZone: null,
    probabilityOfSleeveLengthInHitZone: null,
    probabilityOfThighCircumferenceInHitZone: null,
    probabilityOfTotalLengthInHitZone: null,
    probabilityOfWaistCircumferenceInHitZone: null,
    probabilityOfWristCircumferenceInHitZone: null
  }
}
```

Status: completed
```javascript
{
  notification_type: 'ESTIMATION_STATUS_UPDATED',
  request: {
    age: 45,
    bicep_circumference: 27.323706,
    calf_circumference: 36.243294,
    chest_circumference: 86.315414,
    created_at: 1561530296,
    updated_at: 1561530301,
    error_code: null,
    error_detail: null,
    front_image_url: null,
    front_thumbnail_image_url: null,
    gender: 'male',
    height: 173,
    high_hip_circumference: 74.957825,
    hip_circumference: 91.75341,
    id: 'uniqueRequestId',
    inseam_length: 85.749,
    knee_circumference: 38.093292,
    neck_circumference: 36.066647,
    outseam_length: 108.689705,
    race: null,
    shoulder_width: 43.07665,
    side_image_url: null,
    side_thumbnail_image_url: null,
    sleeve_length: 82.65047,
    backlength: null,
    underbust: null,
    status: 'completed',
    thigh_circumference: 55.897587,
    mid_thigh_circumference: 48.660294,
    total_length: 155.41977,
    user_id: 'uniqueUserId',
    waist_circumference: 71.03106,
    weight: 68,
    wrist_circumference: 16.398588,
    fail_on_automatic_estimation_failure: true,
    probabilityOfBicepCircumferenceInHitZone: 81,
    probabilityOfCalfCircumferenceInHitZone: 81,
    probabilityOfChestCircumferenceInHitZone: 98,
    probabilityOfHighHipCircumferenceInHitZone: 76,
    probabilityOfHipCircumferenceInHitZone: 84,
    probabilityOfInseamLengthInHitZone: 93,
    probabilityOfKneeCircumferenceInHitZone: 93,
    probabilityOfMidThighCircumferenceInHitZone: 94,
    probabilityOfNeckCircumferenceInHitZone: 86,
    probabilityOfOutseamLengthInHitZone: 88,
    probabilityOfShoulderWidthInHitZone: 83,
    probabilityOfSleeveLengthInHitZone: 89,
    probabilityOfThighCircumferenceInHitZone: 96,
    probabilityOfTotalLengthInHitZone: 100,
    probabilityOfWaistCircumferenceInHitZone: 96,
    probabilityOfWristCircumferenceInHitZone: 86
  }
}
```

Status: failed
```javascript
{
  notification_type: 'ESTIMATION_STATUS_UPDATED',
  request: {
    age: 33,
    bicep_circumference: null,
    calf_circumference: null,
    chest_circumference: null,
    created_at: 1561708238,
    updated_at: 1561708242,
    error_code: 'CONFIDENCE_INVALID_FRONT_IMAGE',
    error_detail: null,
    front_image_url: null,
    front_thumbnail_image_url: null,
    gender: 'male',
    height: 162,
    high_hip_circumference: null,
    hip_circumference: null,
    id: 'uniqueRequestId',
    inseam_length: null,
    knee_circumference: null,
    neck_circumference: null,
    outseam_length: null,
    race: 'human',
    shoulder_width: null,
    side_image_url: null,
    side_thumbnail_image_url: null,
    sleeve_length: null,
    backlength: null,
    underbust: null,
    status: 'failed',
    thigh_circumference: null,
    mid_thigh_circumference: null,
    total_length: null,
    user_id: 'uniqueUserId',
    waist_circumference: null,
    weight: 70,
    wrist_circumference: null,
    fail_on_automatic_estimation_failure: true,
    probabilityOfBicepCircumferenceInHitZone: 88,
    probabilityOfCalfCircumferenceInHitZone: 90,
    probabilityOfChestCircumferenceInHitZone: 100,
    probabilityOfHighHipCircumferenceInHitZone: 90,
    probabilityOfHipCircumferenceInHitZone: 77,
    probabilityOfInseamLengthInHitZone: 94,
    probabilityOfKneeCircumferenceInHitZone: 94,
    probabilityOfMidThighCircumferenceInHitZone: 92,
    probabilityOfNeckCircumferenceInHitZone: 92,
    probabilityOfOutseamLengthInHitZone: 92,
    probabilityOfShoulderWidthInHitZone: 75,
    probabilityOfSleeveLengthInHitZone: 79,
    probabilityOfThighCircumferenceInHitZone: 98,
    probabilityOfTotalLengthInHitZone: 100,
    probabilityOfWaistCircumferenceInHitZone: 100,
    probabilityOfWristCircumferenceInHitZone: 77
  }
}
```

### 推定リクエストの作成
推定リクエストは`createEstimationRequest`メソッドで作成できます。
引数として、`estimationParameter`及び`callback`を必要とします。

`estimationParameter`は年齢、身長、体重、性別、正面・側面画像をプロパティに持つオブジェクトです。
いずれも不可欠であり、nullないしundefinedは許容されません。

estimationParameter
```javascript
{
  age: 20,
  heightInCm: 170,
  weightInKg: 60,
  gender: bodybank.Gender.male,
  frontImage: frontFileObj,
  sideImage: sideFileObj
}
```

`callback`には推定リクエスト作成の成功・失敗をハンドリングするための関数を渡します。
推定リクエストの作成時には高画質な画像のアップロードが含まれ、完了までに長時間を要する可能性があるため、callbackで非同期処理を実装する必要があります。

callback
```javascript
const estimationCallback = ({ request, errors }) => {
  if (errors && errors.length) {
    // エラーハンドリング
    errors.forEach((error) => {
      console.log(error)
    })

    return
  }

  if (!request) {
    throw new Error('Request should be returned after creation completed.')
  }

  console.log(request.id) // requestId
  console.log(request.user_id) // userId
}
```

推定リクエスト作成を行うサンプルコードを記載します。

```javascript
function callCreateEstimationRequest() {
  const genderElem = document.getElementById("gender")
  const gender = genderElem.options[genderElem.selectedIndex].value
  let bodybankGender

  switch (gender) {
    case 'male':
      bodybankGender = bodybank.Gender.male
      break;
    case 'female':
      bodybankGender = bodybank.Gender.female
      break;
    case 'none':
      bodybankGender = null
      break;
    default:
      throw new Error('Unexpected gender.')
  }

  const estimationParameter = {
    age: document.getElementById("age").value,
    heightInCm: document.getElementById("height").value,
    weightInKg: document.getElementById("weight").value,
    gender: bodybankGender,
    frontImage: frontElem.files[0],
    sideImage: sideElem.files[0],
    failOnAutomaticEstimationFailure: true
  }
  const callback = ({ request, errors }) => {
    if (errors && errors.length) {
      // エラーハンドリング
      errors.forEach((error) => {
        console.log(error)
      })

      return
    }

    if (!request) {
      throw new Error('Request should be returned after creation completed.')
    }

    console.log(request.id) // requestId
    console.log(request.user_id) // userId
  }

  bodybank.createEstimationRequest({ estimationParameter, callback })
}

callCreateEstimationRequest()
```

上記の通り、callbackの引数として作成完了した推定リクエストを受け取ることができます。

`request`オブジェクトはユニークなidプロパティを保持しています。  
このidプロパティを使って、推定結果をクエリすることができます。

### 推定リクエストの詳細を取得する
特定の推定リクエストのデータを取得するために、`getEstimationRequest`メソッドがあります。

推定結果はこのメソッドで取得できるrequestオブジェクトに含まれています。

本メソッドは`id`及び`callback`を引数に取ります。

`id`は、推定リクエスト作成成功時に取得できるユニークなrequestIdです。  
詳細は前項を参照してください。

```javascript
function getEstimationRequest(requestId) {
  bodybank.getEstimationRequest({
    id: requestId,
    callback: ({ request, errors }) => {
      if (errors && errors.length) {
        // エラーハンドリング
        errors.forEach((error) => {
          console.log(error)
        })

        return
      }

      if (!request) {
        throw new Error('Request should exist.')
      }

      if (request.frontImage) {
        const urlPromise = request.frontImage.downloadableURL
        urlPromise.then(res => {
          const image = document.getElementById("front-image")
          image.src = res
        })
      }
    }
  })
}
```

### 推定リクエストの一覧を取得する
ユーザーがこれまでに行った推定の一覧を取得したい場合には`listEstimationRequests`メソッドを利用します。

本メソッドは下記のようなオブジェクトを引数に取ります。

```javascript
{
  callback, // 取得成功・失敗時に呼ばれるコールバック
  limit, // 1リクエストで取得したい推定リクエストの数
  nextToken // 次回のクエリでどこから推定リクエストを取得すべきかを表すトークン
}
```

下記はリクエスト一覧を取得するための最低限のhtml, css, jsのセットです。

index.html
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Sample</title>
    <link rel="stylesheet" href="css/style.css">
    <script type="text/javascript" src="../bodybank-enterprise.min.js"></script>
  </head>
  <body>
    <button id="show-list">getList</button><br>
    <!-- 縦スクロール可能なエレメント。最下部までスクロールすると次のリクエスト一覧を取得する -->
    <ul id="request-list" onscroll="fetchNext()"></ul>
    <!-- javascriptファイルの読み込み -->
    <script type="text/javascript" src="index.js"></script>
  </body>
</html>
```

css/style.css
```css
#request-list {
  max-height: 80px;
  background-color: aliceblue;
  overflow-y: auto;
}
```

index.js
```javascript
let listNextToken // サーバーから返却されたnextTokenを格納する変数
let nextExists = true // 未取得のリクエストが存在するかどうかを表すフラグ
let isFetchingList = false // 多重のリクエストを送らないようにするためのフラグ

// 推定リクエストのリストを取得するメソッド
function getEstimationRequestsList(nextToken) {
  const callback = ({ requests, nextToken, errors }) => {
    isFetchingList = false

    if (errors && errors.length) {
      // エラーハンドリング
      errors.forEach((error) => {
        console.log(error)
      })

      return
    }

    nextExists = !!nextToken // nextTokenがfalsyであれば、未取得のリクエストは存在しない
    listNextToken = nextToken

    if (requests) {
      const listElem = document.getElementById("request-list")

      requests.forEach(request => {
        var li = document.createElement("li")

        li.setAttribute('id', request.id)
        li.appendChild(document.createTextNode(`${request.id}: ${request.user_id}`))
        listElem.appendChild(li)
      })
    }
  }

  bodybank.listEstimationRequests({
    limit: 5,
    nextToken,
    callback
  })
}

// 初回のリスト取得
document.getElementById("show-list").onclick = () => {
  getEstimationRequestsList(listNextToken)
}

function fetchNext() {
  const listElem = document.getElementById("request-list")
  const {
    offsetHeight,
    scrollTop,
    scrollHeight
  } = listElem
  const isEndOfList = offsetHeight + scrollTop >= scrollHeight

  if (isEndOfList && nextExists && !isFetchingList) {
    isFetchingList = true // スクロールイベントが複数回発火するのを防ぐ
    getEstimationRequestsList(listNextToken)
  }
}
```

### エラーコード
SDKから返される可能性のあるエラー及び詳細の理由は下記のとおりです。
すべてビルトインのErrorクラスを継承しています。

#### CoreError
- UnexpectedError: 想定外エラー
- NotInitialized: SDKが初期化されていない
- NoTokenProvider: tokenProviderが渡されていない
- FailedToInitialize: 何らかの理由でSDK初期化に失敗
- LocalStorageNotFound: localStorageが見つからない (ブラウザはlocalStorageを持っているはず)
- FailedToSignIn: AWSを利用したサインインが失敗

#### EstimationError
- InvalidResponse: Bodygramサーバーから不正なレスポンスが返却された

#### TokenError
- NoRestoreTokenBlock: restoreTokenBlockが定義されていない
- FailedToFetchBodyBankToken: bodybank tokenの取得に失敗
- FailedToRefreshBodyBankToken: bodybank tokenの復元に失敗
- NoUserIdSpecified: DirectTokenProvider利用の際にuserIdが未定義
