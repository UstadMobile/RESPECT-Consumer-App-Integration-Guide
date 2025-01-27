# RESPECT Launcher App Integration Guide

Overview: 

Terms:

__Learning Unit__: A distinct learning unit that can be assigned to a learner. This may or may not involve any assessment. The RESPECT 
launcher app can be used to launch any RESPECT Compatible app itself as well as launch any specific learning unit.

## Steps

### Create a manifest
```
{
   "name": {
      "en-US": "My app"
   },
   license: "(identifier from https://spdx.org/licenses)",
   learningUnits: "https://example.org/opds.json",
   defaultLaunchUri: "https://example.org/",

   android {
       packageId: "org.example.app",
       stores: ["https://play.google.com/store/apps/details?id=org.example.app", "..."]
   }
}
```

* name: a [language map](https://github.com/adlnet/xAPI-Spec/blob/master/xAPI-Data.md#42-language-maps) of the app's name
* license: the license ID for the app itself.
* learningUnits: a link (absolute or relative) to an [OPDS-2.0 catalog](https://drafts.opds.io/opds-2.0.html) of Learning Units
* defaultLaunchUri: the URL that the RESPECT launcher app will use to launch the app if no learning unit is specified.
* android.packageId: package id of the app on Android
* android.stores: List of app store URLs from which the app can be downloaded (e.g. Google Play, F-Droid, etc)

### Bind RESPECT Launcher Service

The RESPECT client manager service ensures that whilst a RESPECT session is ongoing (e.g. there are any active single sign
on sessions or any active learning units launched via the launcher) the RESPECT Launcher App will be kept alive (for it to provide
HTTP APIs).

In your Android Activity:

```
val respectClientManager = RespectClientManager()

fun onCreate(savedInstanceState: Bundle?) {
    respectClientManager.bindService(this)
}
```

### Supporting single sign-on

1) Detect RESPECT Launcher apps installed and display login buttons alongside other single sign-on / social login options
2) If the user selects a RESPECT Launcher single sign on, then
```
val authRequest = RespectSingleSignOnRequest.Builder()
   .addScope("oneroster-scopes")  //OneRoster scopes, if required as per https://www.imsglobal.org/sites/default/files/spec/oneroster/v1p2/rostering-restbinding/OneRosterv1p2RosteringService_RESTBindv1p0.html#Main4p3
   .build()

val result = respectConsumerManager.requestSingleSignOn(authRequest)
  
```  


