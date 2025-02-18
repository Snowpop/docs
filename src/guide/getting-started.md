# Getting started

We will now go into to details, what you need to know about the API and explain the terminology and concepts of the API.

## Introduction

The passwordless.dev service consists of three parts:

* A small open-source client side library. The library helps you interact with the browser WebAuthn API and to connect with the public passwordless API.
* A public API, for browsers to negotiate public keys and cryptographic details with the browser
* A secret API, for your backend server to initiate key registrations, verify sign ins and retrieve keys for your users


## Terminology and concepts

This section will only describe concepts specific to the Passwordless API. See [What is Webauthn and FIDO2?](what-is-webauthn-and-fido2) for more terminology.

When using the Passwordless API, you will encounter three type of tokens:

* **ApiKey** `example:public:6b086b1e...` This is a Public API key, safe and intended to be included client side. It allows the browser to connect to our backend and initiate key negotiations and assertions.
* **ApiSecret** `example:secret:4fd1992...` This is a Secret API key and should be well protected. It allows your backend to verify sign ins and register keys on behalf of your users. ([Create an account](https://www.passwordless.dev/create-tenant/) to get your API keys.)
* **Token** `wWdDh02ItIvnCKT...` This is a ephemeral token (exists only temporarily) and is passed between the client, your backend and the Passwordless API. It encodes the status of an ongoing operation, such as registering a credential or signing in. You can think of it as an session or JWT token.

It's also good to understand how WebAuthn and the Passwordless API treats UserIDs, Emails, usernames, etc.

* **UserId** is a string which will be used as the [WebAuthn Userhandle](https://www.w3.org/TR/webauthn-2/#dom-publickeycredentialuserentity-id). It must **NOT contain PII** such as an email or similiar (This should be a database ID or GUID).
* **Alias** is a "human friendly" reference to a UserId. In order to allow a sign in to be initiated with a "username" of some sort (email, username, phonenumer etc), it is possible (but not required) to attach one or multiple aliases to a specific UserId. This is done with the Alias API endpoint. The Alias is hashed before storage and is only an alternative way to initiate a signin (e.g. when the UserId might not be known to the front end code initiating the sign in)

## ✨ Quick start with copy-paste <Badge text="frontend only" type="tip"/>

If you just want to **try signing in using your face/fingerprint**, you can copy-paste the demo code to your application.

[Get the code](demo-and-examples.html#copy-paste-demo-client-side-only)

## ✅ First step

To use Passwordless you need to add our library to your frontend and add a small backend integration. 
Integrating Passwordless into your app or website can begin as soon as you create a Passwordless account, and requires three steps:


1. [Obtain your API keys](https://www.passwordless.dev/create-tenant/) so Passwordless can authenticate your integration’s API requests.
2. [Install the client library](https://github.com/passwordless/passwordless-client-js#get-coding) so your integration can interact with the Passwordless API and the browser
3. Register a credential to confirm everything is up and running


## Register a credential

This section will explain the flow of this operation. Code is available in the examples section

1. Your backend calls the Passworldess api `/register/token` endpoint with the username/id of the user.
2. Client-side initiates the WebAuthn process and is allowed to store the credential with the Passwordless API using the token.

![Passwordless register flow](https://cdn.passwordless.dev/assets/passwordless.register.png)

#### 1. Retrieve a token <Badge text="backend" type="warning"/>

To register a credential for a user, your backend calls the passwordless api:

<CodeSwitcher :languages="{js:'JavaScript',http:'HTTP'}">
<template v-slot:js>

```js
// your backend app.js
const payload = {
    userId: "123",
    username:"anders@passwordless.dev",
    displayname:"Mr. Anders Åberg",
    aliases: ["anders@passwordless.dev"] // Allow signin to be initiated without knowing userid
};

// Make a HTTPS POST to `/register/token` with the UserId (using your ApiSecret)...
var token = await fetch(apiurl + "/register/token", {
    method: "POST",
    body: JSON.stringify(payload),
    headers: { ApiSecret: API_SECRET, 'Content-Type': 'application/json'}
}).then(r => r.text());

console.log("received token", token); // "register_wWdDh02ItIvnCKT_02ItIvn..."
```
</template>
<template v-slot:http>

```http
POST https://v3.passwordless.dev/register/token
ApiSecret: demo:secret:yyy
Content-Type: application/json

{ "UserId": "123", "username": "anders@passwordless.dev", "displayName": "Mr. Anders Åberg" } 
```
Response:
```json
"register_wWdDh02ItIvnCKT_02ItIvn..."
```
</template>
</CodeSwitcher>

#### 2. Initiate the registration <Badge text="frontend" type="tip"/>

[Get the client-side library](https://github.com/passwordless/passwordless-client-js) from a cdn or npm.
Your client-side code can start the registration progress. The keys will automatically be stored in the passwordless api

```js
var p = new Passwordless.Client({
    apiKey: "demo:public:6b08891222194fd1992465f8668f"
});

// register_wWdDh02ItIvnCKT_02ItIvn...
var myToken = await fetch("/example-backend/passwordless/token").then(r => r.text());

try {
    await p.register(myToken);
    // success!
} catch (e) {
    // error    
}
```


## Sign in

This section will explain the flow of this operation. Code is available in the examples section

1. Client-side library calls the passwordless API with the UserId or Alias and initiates the WebAuthn process.
    * If the sign-in is cryptographically successful, a token is returned from the passwordless api to the client side.
    * Client-side forwards the token your backend.
2. Your backend calls the Passworldess api `/signin/verify` endpoint with the token.
    * The Passwordless API verifies that the signin was successful and returns the verified user and additional information about the sign in. 

![Passwordless sign in flow](https://cdn.passwordless.dev/assets/passwordless.signin.png?v1)

#### 1. Start the passwordless sign in <Badge text="frontend" type="tip"/>

[Get the client-side library](https://github.com/passwordless/passwordless-client-js) from a cdn or npm.
Pass a alias or id to the sign in method to begin the WebAuthn process.

```js
var p = new Passwordless.Client({
    apiKey: "demo:public:6b08891222194fd1992465f8668f"
});

var alias = "anders@user.com"; // get username from input

// returns verify_yUf6_wWdDh02ItIvnCKT_02ItIvn...
var token = await p.signinWithAlias(alias);
// var token = await p.signinWithConditional(); // Uses what is known as the Conditional UI to sign in using autocomplete in the browser
// var token = await p.signinWithId("123"); // if you did not set an alias, you can signin with the UserId.

// verify the token
var verifiedUser = await fetch("/example-backend/passwordless/signin?token=" + token).then(r => r.json());
if(verifiedUser.success === true) {
    //success!
}
```

#### 2. Verify the token <Badge text="backend" type="warning"/>

Once the client-side code has finished the WebAuthn process you need to verify the token with the backend api.
Only then can you trust that the WebAuthn process succeeded and identify what user has signed in.

<CodeSwitcher :languages="{js:'JavaScript',http:'HTTP'}">
<template v-slot:js>

```js
const token = { token: req.query.token };

const response = await fetch(apiurl + "/signin/verify", {
    method: "POST",
    body: JSON.stringify(token),
    headers: { ApiSecret: API_SECRET, 'Content-Type': 'application/json' }
});

var body = await response.json();
if (body.success) {
    console.log("Succesfully verfied signin for user", body);
} else {
    console.warn("Sign in failed", body);
}
```
</template>
<template v-slot:http>

```http
POST https://v3.passwordless.dev/signin/verify
ApiSecret: demo:secret:yyy
Content-Type: application/json

{ "token": "verify_yUf6_wWdDh02ItIvnCKT_02ItIvn..." }
```
</template>
</CodeSwitcher>

Response:
```json
{
  "success": true,
  "userId": "123",
  "timestamp": "2021-08-01T01:33:36.9773187Z",
  "rpid": "example.com",
  "origin": "http://example.com:3000",
  "device": "Chrome, Windows 10",
  "country": "",
  "nickname": "Home laptop",
  "credentialId": "Mq1ZhrHBmhly34YaO/uuXuNuf/VCHDkuknENz/LZJR4=",
  "expiresAt": "2021-08-01T01:35:36.9773193Z"
}
```

## And you're done!

That's all that is required to get started with Passwordless.
There will be more to explore however, see our other API endpoints.


## Using Autofill / Conditional UI

The browser can present webauhtn credentials in input fields, making it easy to migrate to using passkeys.
