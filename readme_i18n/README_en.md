# Bodygram SDK JS Readme
Bodygram is a system for estimating body size of human with a few parameters.  
You throw a "body size estimation request" (which we'll call just an "estimation request") to our server via this SDK, and after some seconds you'll receive an estimation result asyncrounously.

> Note: This SDK is developed assuming usecase on front-end, not server.  

> Note: This SDK is made for modern mobile use, assuming ES6 (ECMAScript 2015) is supported.

This SDK takes care of...

1. Creation & Restoration of token
2. Creation of new estimation requests
3. Getting result of estimation requests

> Note: NO UI IS PROVIDED BY THIS SDK.  

## Version
2019-06-30 SDK Version 0.0.1

## Browser Coverage
[browserlist](https://browserl.ist/?q=%3E1%25+and+not+ie+%3C%3D+11+and+not+op_mini+all)

## Installation
Clone this repository, and put `bodybank-enterprise.min.js` in your project.
In your index html file, read the script above as below.

```html
<head>
  <meta charset="UTF-8">
  <title>Your awesome website</title>
  <script type="text/javascript" src="./bodybank-enterprise.min.js"></script>
</head>
```

## Usage
### Set up
Include the `bodybank-config.json` file into your project.  
The file is provided after making a contract.

### Token Provider 
Token Provider is a class which does issueing and managing of token, which is for accessing Bodygram's backend resource. We have 2 kinds of tokenProvider.

- Default Token Provider
- Direct Token Provider

The former is for production, another is for development environment.

We'll see both of them from now on.

#### Direct Token Provider
First explanation is for Direct Token Provider, that is for development env.

Because this functionality is for only development, please do implement with Default Token Provider before you release on production.

`DirectTokenProvider` is a descendant of `DefaultTokenProvider`, and it's easened to use.

```javascript
const bodybank = new BodyBankEnterprise()
const tokenProvider = bodybank.getDirectTokenProvider("https://api.<SHORT IDENTIFIER>.enterprise.bodybank.com", "YOUR API KEY")

tokenProvider.tokenDuration = 86400
tokenProvider.userId = "encryptedUserId"
```

Just a few line of codes are required to setup.

#### Default Token Provider
Next is for Default Token Provider for production use.

When you develop with it, the step will be like...

1. Create a new endpoint and implement the process of fetching token.
2. From the project in which you use Bodygram, call the endpoint you've created.

Detail below.

Assume that you have an API server on `https://awesome-api.com`.  
There might be an endpoint for fetching Bodygram token, which can be called by GET request to `https://awesome-api.com/bodygram/token`.

The endpoint implementation might be look like below.
(Example is made by Node.js and Express.)

```javascript
const rp = require('request-promise')

export default async function (req, res) {
  const apiKey = "YOUR API KEY"
  const apiEndpoint = "https://api.<SHORT IDENTIFIER>.enterprise.bodybank.com"
  const userId = 'encryptedUserId' // Fixed value for easy
  const options = {
    headers: { "x-api-key": apiKey },
    json: { user_id: userId }
  }

  const tokenRes = await rp.post(apiEndpoint, options)
  
  res.send(tokenRes.content.token);
}
```

You'll throw POST request to the `apiEndpoint` with `apiKey` and `userId`.

When it's done, go back to front-end project and implement below.

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

`restoreTokenBlock` is a function which is called whenever a token is nearly expired or expired.

You have to prepare your own api for fetching Bodygram token so as not to expose your `apiEndpoint` and `apiKey` to malicious outsiders.

### SDK initialization
Initialize BodyBankEnterprise before creating request.  
You need to pass tokenProvider and bodybank-config.

```javascript
bodybank.initialize(tokenProvider, bodybankConfig)
```

### Use webhook to subscribe to the process of estimation requests
Estimation requests are processed asynchronously.  
To catch up with the status of estimation requests, we provide webhook functionality.

In order to use this, please tell us the API endpoint you want to be called after the estimation requests' status changed and also api key for authorization.

Right now you can set only 1 endpoint.

The endpoint will be called when the estimation requests' status becomes...

1. requested (estimation request is created)
2. requested -> pendingAutomaticEstimation (waiting for estimation)
3. pendingAutomaticEstimation -> completed
4. requested or pendingAutomaticEstimation -> failed

We use POST method so that we can pass on estimation requests in request body.

Below are examples of request body on each status.

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
    id: 'requestId',
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
    user_id: 'userId',
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
    id: 'requestId',
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
    user_id: 'userId',
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
    id: 'requestId',
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
    user_id: 'userId',
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

### Create estimation request
Make estimation request by `createEstimationRequest` method.
It takes `estimationParameter` and `callback` as arguments.

`estimationParameter` includes age, height, weight, gender, frontImage, and sideImage.
Any of them is essential and not allowed to be null or undefined.

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

`callback` is for handling success or error.  
Creating estimation request includes uploads of high resolution images, so it might take a long time to be complete.

callback
```javascript
const estimationCallback = ({ request, errors }) => {
  if (errors && errors.length) {
    // Do some error handling here.
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

Below is a sample which is implementing creation of estimation request.

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
      // Do some error handling here.
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

As you can see, you'll receive created estimation request as an argument of callback, and the request object has `id` property which is unique.

We recommend to save the id in your database ASAP after receiving it, for it's necessary to get the result of estimation.

### Get estimation request
In order to get particular estimation request's data, you can call `getEstimationRequest` method.

Estimation result is included in the request object you get from this method.

This function takes `id` and `callback` as arguments.

`id` is a requestId which you can get after success of creating an estimation request. (Please see the code example above.)

```javascript
function getEstimationRequest(requestId) {
  bodybank.getEstimationRequest({
    id: requestId,
    callback: ({ request, errors }) => {
      if (errors && errors.length) {
        // Do some error handling here.
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

### Get estimation requests list
Sometimes you want to get all requests the user made so far.
For such needs, we have `listEstimationRequests` method.

It takes an object like below as an argument.

```javascript
{
  callback, // called on success or error
  limit, // number of requests to be fetched
  nextToken // string which indicates from where to query for the next search
}
```

Below is just a little example (html, css, js) for fetching requests list.

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
    <!-- Scrollable element. When scrolled to the bottom, tries to fetch another bunch of requests. -->
    <ul id="request-list" onscroll="fetchNext()"></ul>
    <!-- Below is for javascript file -->
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
let listNextToken // Variable for storing nextToken got from server
let nextExists = true // Flag that indicates whether there're unfetched requests
let isFetchingList = false // Flag to suspend making duplicate request

// function for getting estimation requests' list
function getEstimationRequestsList(nextToken) {
  const callback = ({ requests, nextToken, errors }) => {
    isFetchingList = false

    if (errors && errors.length) {
      // Do some error handling here.
      errors.forEach((error) => {
        console.log(error)
      })

      return
    }

    nextExists = !!nextToken // If you don't receive nextToken, there won't be extra requests anymore
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

// fetch first 5 requests
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
    isFetchingList = true // If not set, scroll event fires multiple times and multiple requests sent.
    getEstimationRequestsList(listNextToken)
  }
}
```

### Error codes
All errors and detail reasons that might be returned from SDK are listed below.
Every Error is extended from built-in Error class.

#### CoreError
- UnexpectedError: Unexpected error occurred
- NotInitialized: BodyBankEnterprise is not initialized
- NoTokenProvider: tokenProvider is not supplied to BodyBankEnterprise
- FailedToInitialize: Error while initializing BodyBankEnterprise

#### EstimationError
- InvalidResponse: Invalid response from bodygram server

#### TokenError
- NoRestoreTokenBlock: restoreTokenBlock is not supplied
- FailedToFetchBodyBankToken: Error while fetching bodybank token
- FailedToRefreshBodyBankToken: Error while refreshing bodybank token
- NoUserIdSpecified: No userId is specified for DirectTokenProvider
