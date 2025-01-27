# RESPECT Launcher App Integration Guide

Overview: The RESPECT Launcher App allows teachers and students to sign in once to easily access any compatible app and control their data. It
is intended to be an open-source alternative to proprietary commercial solutions such as [Clever](https://clever.com), 
[ClassLink Launchpad](https://play.google.com/store/apps/details?id=com.classlink.launchpad.android&hl=en), and [Wonde](https://wonde.com/) that
works offline as well as online.

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

The manifest is used by the RESPECT launcher app to enable administrators to easily add a new app by simply copy/pasting a link. It is
recommended that SHOULD be https://example.org/.well-known/respect-app.json ; such that a user can simply use 'example.org'.

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

### Use the libRESPECT Proxy Cache

The RESPECT Launcher app MAY send your app a signal to download a specific Learning Unit for offline use. This
can be handled using the libRESPECT cache and service. 

1) Add the service to AndroidManifest.xml:
```
<service
      android:name="world.respect.librespect.clientdownloadservice"
      android:enabled="true"
      android:exported="true">

<intent-filter>
    <action android:name="respect.world.librespect.ACTION_CLIENT_LU_DOWNLOAD" />
    <action android:name="respect.world.librespect.ACTION_CLIENT_LU_RELEASE" />
</intent-filter>
</service>
```

2) Use the libRESPECT HTTP Cache in your Android application where you create an OKHTTP instance:

```
val libRespectCache = LibRespectCacheBuilder.build()
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(LibRespectCacheInterceptor(libRespectCache))
    .build()

//OR use the proxy
val httpProxy = LibRespectProxyServer(libRespectCache)
httpProxy.start() //Now use: httpProxy.listeningPort as the HTTP proxy
```


### Support single sign-on option

1) Detect RESPECT Launcher apps installed and display login buttons alongside other single sign-on / social login options
2) If the user selects a RESPECT Launcher single sign on, then
```
val authRequest = RespectSingleSignOnRequest.Builder()
   .addScope("oneroster-scopes")  //OneRoster scopes, if required as per https://www.imsglobal.org/sites/default/files/spec/oneroster/v1p2/rostering-restbinding/OneRosterv1p2RosteringService_RESTBindv1p0.html#Main4p3
   .build()

val result = respectConsumerManager.requestSingleSignOn(authRequest)
/*
 * Result includes:
 *  result.userId an arbitrary user id string. Anonymization / privacy settings may result in this string being different for the same user
 *  result.displayName an arbitrary string. This may be the first name of the user or used on greetings
 *  result.token.bearer (always a Bearer token). Add this to http requests e.g. Authorization: Bearer <token>
 *  result.endpoints.xapi.url : The xAPI endpoint URL (will be on localhost).
 *  result.endpoints.oneroster.url : The URL of the OneRoster endpoint (will be on localhost)
 */  
```  

Once the user has signed in, the client app can use the HTTP APIs to save and retrieve learner data, progress information, etc.

### Supporting launching a specific Learning Unit

Launching a specific Learning Unit is based on normal [app links](https://developer.android.com/training/app-links). The app links must be verified as usual.

When the respectLaunchVersion parameter is included, the client app MUST use the provided endpoints to send progress information back to the RESPECT
laucnher app and then finish (exit) the activity when the learning unit is complete.

The URI launched by the RESPECT launcher will be the Learning Unit to be completed with the following additional parameters:
* respectLaunchVersion=1 Indicates that the launch is coming from a RESPECT Launcher app
* auth : The authentication to use with [xAPI](https://www.xapi.com) and/or [AGS](https://www.imsglobal.org/spec/lti-ags/v2p0/) API. 
* Some or none of the [OpenID Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), e.g. given_name, locale, sub,
  depending on the privacy settings being used by the RESPECT Launcher.
* http_proxy : an HTTP Proxy provided by the RESPECT launcher app. This proxy MAY download URLs required by the Learning Unit (e.g. when a student is
  assigned a given Learning Unit).   
* endpoint_lti_ags : the HTTP url to use for the [LTI Assignment and Grade Services Specification](https://www.imsglobal.org/spec/lti-ags/v2p0/)
* endpoint : the HTTP url to use for xAPI ( named 'endpoint' because this is as per the Rustici launch spec, see below)
* All of the [Rustici Launch Parameters](https://github.com/RusticiSoftware/launch/blob/master/lms_lrs.md) as per the xAPI spec.

E.g. where a Learning Unit ID is https://example.org/topic/learningUnit1, then :
```
https://example.org/topic/learningUnit1/?respectLaunchVersion=1&auth=[secret]&given_name=John&locale=en-US
   &http_proxy=http://localhost:8098/
   &endpoint_lti_ags=http://localhost:8097/api/ags
   &endpoint=http://localhost:8097/api/xapi
   &actor={ "name" : ["Project Tin Can"], "mbox" : ["mailto:tincan@scorm.com"] }   
   &registration=760e3480-ba55-4991-94b0-01820dbd23a2   
   &activity_id=https://example.org/topic/learningUnit1/
```
Each Learning Unit MUST link to a manifest containing a list of all the URLs that need to be available for the Learning Unit
to run. This allows the RESPECT Launcher App to download these in advance where needed (e.g. if a student is assigned to
complete the given Learning Unit).

This can be done one of two ways:

* Include a link in the HTML returned by the Learning Unit ID as per:
```
<link rel="https://respect.world/ns/learning-unit-urls" type="text/plain" href="urls.txt"/>
```
* Use the .well-known path e.g.
```
https://https://example.org/topic/learningUnit1/.well-known/respect-urls.txt
```
