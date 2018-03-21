---
layout: post
title:  "Facebook OAuth using SFAuthenticationSession on iOS 11"
author: "Rohan Nagar"
---

In many applications you write, you will want to connect to a user's Facebook account to either
create posts or read their information. The way to do this is with [OAuth 2](https://oauth.net/2/), which provides
a way for Facebook to give your application a specific token for each user. Your application can then
use that token to communicate with the Facebook APIs on the user's behalf. When writing a Swift application
for iOS 11, there are two ways to do this:

1. Use the [Official Facebook Swift SDK](https://developers.facebook.com/docs/swift/)
2. Use [SFAuthenticationSession](https://developer.apple.com/documentation/safariservices/sfauthenticationsession)
with an authentication URL

The first one is simple enough, and for many applications you **will** want to use the SDK so that you can
easily call other Facebook API methods from within your app. However, in some applications you may want
to **only** get the token and then pass it to your backend that does all of the work interacting with
Facebook. This is the model that my colleague and I use for [Pilot](https://github.com/SanctionCo/pilot-ios).
For these situations, it doesn't make sense to import an entire SDK just for getting the OAuth token. And in
fact, `SFAuthenticationSession` makes it dead simple to do this yourself.

Let's see how it's done.

# Facebook Developer Setup

Even if you've already set up your app with Facebook, make sure you go through this section to ensure that
you've set up the bundle ID in the settings correctly.

First, you will want to register an application with Facebook. You can do this by going to
[the developer portal](https://developers.facebook.com), logging into your Facebook account, and choosing
the option in the top right to `Add a New App`.

![Developer Portal](/assets/img/facebook-oauth/developer_portal.png)

Enter in your application name and provide a contact email.

![Create App](/assets/img/facebook-oauth/create_app.png)

<br>
When the app is created, you will be taken to the dashboard for your application. Go to `Settings` in the left
sidebar. Note your `APP ID` in the top.

![Facebook Settings](/assets/img/facebook-oauth/facebook_settings.png)

<br>
Scroll down the settings page to see the button `+ Add Platform`. Click the button and choose `iOS` as the platform
type.

![Add Platform](/assets/img/facebook-oauth/add_platform.png)

<br>
Finally, in the `Bundle ID` section for the iOS platform, enter the letters `fb` followed by your App ID number. For
example, if my App ID is `470209193334943`, then I will enter the value `fb470209193334943`.

![Bundle ID](/assets/img/facebook-oauth/bundle_id.png)

<br>
Save your changes, and now you're ready to go into XCode!

# Create a URL Scheme

Open your project in Xcode 9.0 or higher. Navigate to the project settings by clicking your project name
at the top of the left sidebar. Once there, go to the `Info` tab for the main app target.

![Xcode Info](/assets/img/facebook-oauth/xcode_info.png)

<br>
Expand the `URL Types` at the bottom, and click the `+` button to create a new URL scheme. For the `URL Schemes`
section, enter the same value as you used for the Facebook bundle ID. So, for example, `fb470209193334943://`.
Leave the rest of the fields blank or default.

![URL Scheme](/assets/img/facebook-oauth/url_scheme.png)

<br>
That's all the setup there is! Finally, we can do some programming.

# Write the Code

Before we write the code to handle the OAuth flow, let's add an extension to the `URL` class to make
our lives easier. Create a new Swift file called `URL+Fragement.swift` and add the following lines:

{% highlight swift  linenos %}
extension URL {

  func getFragementParam(key: String) -> String? {
    if let params = self.fragment?.components(separatedBy: "&") {
      for param in params {
        if let value = param.components(separatedBy: "=") as [String]? {
          if value[0] == key {
            return value[1]
          }
        }
      }
    }
        
    return nil
  }

}
{% endhighlight %}

Let's go over this in more detail. After a user authenticates with Facebook, Facebook calls a redirect URL that you specifiy
in order to redirect the user back to your application. When the URL is called, Facebook adds a few
fragement parameters. For example, if your redirect URL is `fb470209193334943://authorize`,
then the URL may be called like this: `fb470209193334943://authorize#access_token=9999`.
In order to more easily extract the value of the `access_token` parameter, we use the function above.

In the `getFragementParam()` function above, we first take the `fragement` of the URL and seperate it by the
character `&`. Then, we iterate over each of these strings (which look like `key=value`) and seperate them by
the character `=`. The `value` variable has two members, at index 0 is the key and at index 1 is the value.
So after we get those split, we can simply check to see if `value[0]` is the fragement parameter that we are
looking for, and if so, return `value[1]`.

Now we're ready to write our OAuth code! First, we need to go to the class where we will kick off the OAuth flow
and create a static member:

{% highlight swift  linenos %}
static var authSession: SFAuthenticationSession? = nil
{% endhighlight %}

This is important because if we create `authSession` as a local variable in the function that we want the
authentication session to start, the object will simply be deallocated after the function completes and the
session will briefly pop up on the screen and then disappear. As a class member, the variable will stick around.

Now go to the location in this class where you want to kick off the OAuth flow. This could be in a function
that handles a button press, for example. Paste the following code (**note that you will need to modify the first
two lines**):

{% highlight swift  linenos %}
let authURL = "https://www.facebook.com/dialog/oauth?client_id=470209193334943&redirect_uri=fb470209190004943%3A%2F%2Fauthorize&scope=public_profile&response_type=token"
let callbackURL = "fb470209193334943://authorize"

self.authSession = SFAuthenticationSession(url: authURL, callbackURLScheme: callbackURL) {
  (callBack: URL?, error: Error?) in
    guard error == nil, let successURL = callBack else {
      // Log error or display error to the user here
      return
    }

    let token = successURL.getFragementParam(key: "access_token")

    // Use your token! Send it to your backend or store it on device
}

self.authSession?.start()
{% endhighlight %}

This code above is where all the magic happens. First, we define two URLs. The `authURL` is the url that
you will send the user to. This is a Facebook URL that contains as query parameters:

1. Your App ID (`client_id`). Set this to your App ID.
2. The redirect URI for Facebook to call when the user is done authenticating (`redirect_uri`). Set this to the
URL scheme you set up for your project, at the path `authorize`. Facebook only accepts the `authorize` path
for custom URL schemes. So, yours will look something like `fb470209193334943://authorize`.
3. The scope of permissions requested (`scope`). Set this to the permissions that you want. Here, I've set it to
just allow the app to access public profile information.
4. The type of response you want from Facebook (`response_type`). Set this to `token` so that you get an OAuth
token back.

The `callbackURL` is the same one that you set as the query param for the Facebook `redirect_uri`. This will
be passed in to the `SFAuthenticationSession` object so that it knows the authentication flow is done when
that URL is called in the browser.

Next, we create our `SFAuthenticationSession` object. We pass in the two URLs we defined. The `url` parameter is the
one to show to the user, and the `callbackURLScheme` is the redirect URL. We also define a closure that will be called
when the `SFAuthenticationSession` completes (when either the redirect URL is called or the user cancels).

In the closure, we first check for any errors (for example, if the user canceled at any point) and return if we find those
errors. Then, we use our helper function to get the access token from the URL that Facebook called. Remember, Facebook
calls our redirect URL with the fragement parameter `access_token`, which is what we look for here. At this point in
the closure, we will have our token! You can send the token to your backend or do anything you want with it.

Finally, we start the authentication session by calling the `start()` function on the object.

And there you have it! Now you can run the app, perform the action that will call your new OAuth code, and see the session
start up.

# Conclusion

The new `SFAuthenticationSession` class in Swift 4 provides a very simple way to authenticate,
not only with Facebook but with any third-party web service. It's secure and requires only a few
lines of code. If all you need to do in your iOS app is get a user tokens and pass them
to your backend, this should be the way to go.

Happy authenticating!

## References

- [SFAuthenticationSession](https://developer.apple.com/documentation/safariservices/sfauthenticationsession)
- [iOS 11, Privacy and Single Sign On by Jordan Morgan](https://medium.com/the-traveled-ios-developers-guide/ios-11-privacy-and-single-sign-on-6291687a2ccc)

