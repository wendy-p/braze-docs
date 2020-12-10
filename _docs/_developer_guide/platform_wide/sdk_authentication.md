---
nav_title: SDK Authenticaton
page_order: 5
hidden: true
description: "Verify the identity of SDK requests"
platform:
  - ios
  - android
  - web
---

# SDK Authentication

SDK Authentication ensures that no one can impersonate or create Braze users using your SDK API Keys.

{% alert important %}
This feature is in _beta_ and only available for Android, Web, and iOS SDKs.
{% endalert %}

## How It Works

Braze SDKs, like other client-side libraries, use a public API key to identify your app. In order to ensure that Braze only trusts requests from authenticated users, this feature allows _you_ to supply Braze with secure, cryptographic proof. After all, your server is the only one who would know whether a user is who they claim to be in your app (using cookies or other persistent session data). 

When enabled, this feature will prevent unauthorized access to send or receive data using your app's SDK API Keys, including:

* Sending custom events, attributes, purchases, and session data
* Creating new users in your Braze App Group
* Updating standard user profile attributes
* Receiving or triggering messages

## Getting Started

There are three steps to get started:

### Step 1: [Server-side Integration][1]

Generate a public and private key-pair, and use your private key to create a JWT (_JSON Web Token_) for the currently logged-in user. 

### Step 2: [SDK Integration][2]

Enable this feature in your app and supply the Braze SDK with a JWT Token generated from [in previous step][1].

### Step 3: [Enabling in the Braze Dashboard][3]

Control this feature's enforcement within the Braze Dashboard. This can be done on an app-by-app basis, so you can roll out implementation on an individual SDK (ios, android, or web) instead of all at once.

## Server-Side Integration {#server-side-integration}

### Generate a Public/Private Key-pair {#generate-keys}

First, generate a public/private key-pair, and keep your Private Key secure. The Public Key will be pasted into the Braze Dashboard in a later step.

{% alert warning %}
Remember! Keep your private keys _private_. Never expose or hard-code your _private_ key in your app or website. Anyone who knows your private key can impersonate or create users on behalf of your application.
{% endalert %}

### Sign the user's JSON Web Token {#create-jwt}

Once you have your _private key_, your server-side application should use it to sign and return a JWT to your app or website for the currently logged-in user.

Typically, this logic could go wherever your app would normally request the current user's profile. Such as a login endpoint, or however your app refreshes the current user's profile.

**JWT Header**

|Field|Required|Description|
|-----|-----|-----|
|`alg`|**Yes**|The supported algorithm is `RS256`.|
|`typ`|**Yes**|The type should equal `JWT`.|
{: .reset-td-br-1 .reset-td-br-2 .reset-td-br-3}

**JWT Payload**

|Field|Required|Description|
|-----|-----|-----|
|`sub`|**Yes**|The "subject" should equal the User ID you supply Braze SDKs when calling `changeUser`|
|`exp`|**Yes**|The "expiration" of when you want this token to expire.|
|`aud`|No|The "audience" claim is optional, and if set should equal "braze"|
|`iss`|No|The "issuer" claim is optiona, and if set should equal your SDK API Key.
{: .reset-td-br-1 .reset-td-br-2 .reset-td-br-3}

### JWT Libraries

To learn more about JSON Web Tokens, or to browse the [many open source libraries](https://jwt.io/#libraries-io) that simplify this signing process, check out [https://jwt.io](https://jwt.io).

## SDK Integration {#sdk-integration}

### Enable this feature in the Braze SDK.

When this feature is enabled, Braze SDKs will append the current user's last known JWT to network requests made to Braze Servers.

Don't worry! You can always disable this feature from the Braze Dashboard, even if the SDK has this enabled.

{% tabs %}
{% tab Javascript %}
When calling `appboy.initialize`, set the optional `sdkAuthentication` property to `true`.

```javascript
appboy.initialize('YOUR-API-KEY-HERE', {
    baseUrl: "YOUR-SDK-ENDPOINT-HERE",
    sdkAuthentication: true
});
```
{% endtab %}
{% tab Java %}

When configuring the Appboy instance, call `setIsSdkAuthenticationEnabled` to `true`.

```java
AppboyConfig.Builder appboyConfigBuilder = new AppboyConfig.Builder()
    .setIsSdkAuthenticationEnabled(true);
Appboy.configure(this, appboyConfigBuilder.build());
```
{% endtab %}
{% tab Obj-C %}
todo
{% endtab %}
{% endtabs %}

### Set the current user's JWT Token

Whenever your app calls the Braze SDK's `changeUser` method, also supply the JWT token that was [generated server-side][4].

You can also update the token at any point in the future, for example: to avoid a token from expiring mid-session.

{% tabs %}
{% tab Javascript %}

Supply the JWT Token when calling `appboy.changeUser`:

```javascript
appboy.changeUser("NEW-USER-ID", {
  sdkAuthenticationToken: "JWT-TOKEN-FROM-SERVER"
});
```

Or, when you have refreshed the user's token mid-session:

```javascript
appboy.setAuthenticationToken("NEW-JWT-TOKEN-FROM-SERVER");
```

{% endtab %}
{% tab Java %}

Supply the JWT Token when calling `appboy.changeUser`:

```java
Appboy.getInstance(this).changeUser("NEW-USER-ID", "JWT-TOKEN-FROM-SERVER");
```

Or, when you have refreshed the user's token mid-session:

```java
Appboy.getInstance(this).setSdkAuthenticationSignature("NEW-JWT-TOKEN-FROM-SERVER");
```
{% endtab %}
{% tab Obj-C %}
todo
{% endtab %}
{% endtabs %}


### Register a callback function for invalid tokens {#sdk-callback}

When this feature is being enforced, the following scenarios will cause SDK requests to be rejected by Braze:

- JWT has expired
- JWT was empty or missing
- JWT failed to verify for the Public Keys you uploaded to the Braze Dashboard

When the SDK requests fail for one of these reasons, Braze SDKs will invoke a callback function you supply, in addition to retrying the failed requests periodically.

To fix any invalid tokens and continue to successfully sync data to Braze, your app should request a new JWT from your server and supply Braze's SDK with this new valid token.

{% tabs %}
{% tab Javascript %}
```javascript
appboy.onSdkAuthenticationFailure(async (errorEvent) => {
  const updated_jwt = await getNewTokenSomehow(errorEvent);
  appboy.setAuthenticationToken(updated_jwt);
});
```
{% endtab %}
{% tab Java %}
```java
Appboy.getInstance(this).subscribeToSdkAuthenticationFailures(errorEvent -> {
    String newToken = getNewTokenSomehow(errorEvent);
    Appboy.getInstance(getContext()).setSdkAuthenticationSignature(newToken);
});
```
{% endtab %}
{% tab Obj-C %}
todo
{% endtab %}
{% endtabs %}

The `errorEvent` data passed to this callback will contain the following information:

TODO: 

|Property|Description|
|--------|-----------|
|reason|A description of why the request failed.|
|error_code|An internal error code used by Braze.|
|user_id|The user ID from which the request failed.|
|signature|The JWT which failed.|
{: .reset-td-br-1 .reset-td-br-2}


## Enabling in the Braze Dashboard {#braze-dashboard}

Once your [Server-side Integration][1] and [SDK Integration][2] are complete, you can begin to enable this feature for those specific apps.

Keep in mind, SDK requests will continue to flow as usual _until_ you set an app's Authentication setting to **required**. 

Should anything go wrong with your integration (i.e. your app is incorrectly passing tokens to the SDK, or your server is generating invalid tokens), simply **disable** this feature in the Braze Dashboard and data will resume to flow as usual, without verification.

### Enforcement Options {#enforcement-options}

Within the Braze Dashboard's `App Settings` page, each app has three SDK Authentication states which control how Braze verifies requests.

![dashboard][8]

|Setting|Description|
|---------|-----|
|**Disabled**|Braze will not verify the user's authenticity. (Default Setting)|
|**Optional**|Braze will verify requests for logged-in users, but will only report on failures through [Currents][5]. No requests will be rejected for invalid tokens.|
|**Required**|Braze will verify requests for logged-in users, and will reject requests made with invalid tokens. Errors will also be sent to [Currents][5].|
{: .reset-td-br-1 .reset-td-br-2}

The "**Optional**" setting is a useful way to monitor the potential impact this feature will have on your app's SDK data when it comes to implementation correctness, as well as monitoring older app versions from before you had implemented this feature. 

Invalid JWT signatures will be reported in both **Optional** and **Required** states, however only the **Required** state will reject SDK requests causing apps to retry and request new signatures.

For more information on monitoring failed requests, see the [Currents Schema][5] for this feature.


## Frequently Asked Questions {#faq}

#### Which SDKs support Authentication? {#faq-sdk-support}

Braze Web SDK (vX.Y), Android SDK (vX.Y), and iOS SDK (vX.Y) include support for this feature.

#### Can I use this feature on only some of my apps? {#faq-app-by-app}

Yes, this feature can be enabled for specific apps and doesn't need to be used on all of your apps.

#### What happens to users who are still on older versions of my app? {#faq-sdk-backward-compatibility}

When you begin to enforce this feature, requests made by older app versions will be rejected by Braze and retried by the SDKs. Once users upgrade their app to a supported version, those enqueued requests will begin to be accepted again.

If possible, you should push users to upgrade as you would for any other mandatory upgrade. Alternatively, you can keep the feature ["optional"][6] until you see that an acceptable percentage of users have upgraded.

#### What expiration should I use when generating JWT tokens? {#faq-expiration}

We recommend using the higher value of: average session duration, session cookie/token expiration, or the frequency at which your application would otherwise refresh the current user's profile. 

#### What happens if a JWT expires in the middle of a user's session?

Should a user's token expire mid-session, the SDK has a [callback function][7] it will invoke to let your app know that a new JWT token is needed to continue sending data to Braze.

#### What happens if my server-side integration breaks and I can no longer create JWTs?

If your server is not able to provide JWT tokens or you notice some integration issue, you can always disable the feature in the Braze Dashboard.

Once disabled, any pending failed SDK requests will eventually be retried by the SDK, and accepted by Braze.

#### Why does this feature use Public/Private keys instead of Shared Secrets?

When using Shared Secrets, anyone with access to that shared secret (i.e. the Braze Dashboard page) would be able to generate tokens and impersonate your end users.

Instead, we use Private Keys so that not even Braze Employees (let alone your own Dashboard users) could possibly discover your Private Keys.

[1]: #server-side-integration
[2]: #sdk-integration
[3]: #braze-dashboard
[4]: #create-jwt
[5]: #todo
[6]: #enforcement-options
[7]: #sdk-callback
[8]: {% image_buster /assets/img/sdk-auth-settings.png %}
