# RESPECT Launcher App Integration Guide

Overview: 

The RESPECT Launcher App allows teachers and students to sign in once to easily access any compatible app and keep control their personal data. It
is intended to be an open-source alternative to proprietary commercial solutions such as [Clever](https://clever.com), 
[ClassLink Launchpad](https://play.google.com/store/apps/details?id=com.classlink.launchpad.android&hl=en), and [Wonde](https://wonde.com/) that works offline as well as online.

Student information and progress/usage data is accessed and stored using the [Experience API (xAPI)](https://www.xapi.com) and 
[OneRoster](https://www.1edtech.org/standards/oneroster).

Apps are expected to support two use cases:
* **User launches a specific Learning Unit from the RESPECT Launcher app**: the RESPECT launcher app is used to launch a specific Learning Unit (e.g. lesson, assessment)
* **User launches the edtech app and logs in via RESPECT (single sign-on)**: the user launches an edtech app (Edtech API consumer app as below) via their operating system's default launcher and selects to login using their RESPECT launcher account

Terms:

* __Learning Unit__: A distinct learning unit that can be assigned to a learner. A Learning Unit can be any 
experience that a learner can complete for educational purposes (e.g. a lesson, assessment, simulation, etc). Some apps (e.g. a utility app used to manage school transport) would not be expected to have learning units.
* __Edtech API consumer app__: An edtech app that targets the APIs as referenced in this guide.

Testing notes:

* Before library availablity, apps can be tested using any existing implementation of the Experience API and OneRoster APIs. If an API is not used by an edtech API consumer app (e.g. OneRoster for apps that do not use enrolment information), the parameter can be ignored.

## Steps

### 1 Create app manifest
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
* learningUnits: a link (absolute or relative) to an [OPDS-2.0 catalog](https://drafts.opds.io/opds-2.0.html) of Learning Units (see Step 5)
* defaultLaunchUri: the URL that the RESPECT launcher app will use to launch the app if no learning unit is specified.
* android.packageId: package id of the app on Android
* android.stores: List of app store URLs from which the app can be downloaded (e.g. Google Play, F-Droid, etc)


**Expected outputs**
* RESPECT Manifest file (as above)


### 2 Add libRESPECT

Add the dependency to Gradle (when available).

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

**Expected outputs**
* App built with librespect dependency (when available)

### 3 Use the libRESPECT Proxy Cache

The RESPECT Launcher app MAY send your app a signal to download a specific Learning Unit for offline use. This
can be handled using the libRESPECT cache and service. 

1) Add the service to AndroidManifest.xml:
```
<service
      android:name="world.respect.librespect.CacheProxyService"
      android:enabled="true"
      android:exported="true">
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

**Expected outputs**
* App built and using http proxy for all http requests
  
### 4 Support single sign-on option

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

**Expected outputs**
* App built; single sign on is displayed when the RESPECT launcher app is installed. User can sign-in using RESPECT launcher account.
* When RESPECT launcher account is active, then learner progress and results are saved using xAPI and/or oneroster parameters as specified in the auth resut.

### 5 Support listing and launching learning units

A Learning Unit can be any distinct educational experience a learner can undertake (e.g. a lesson, assessment, simulation, etc). It has a manifest which lists all the resources required (e.g. so they can be downloaded in advance for the Learning Unit to be used offline later).

**Learning Unit Listing:**

If an app contains Learning Units it MUST provide an OPDS catalog listing of those Learning Units where:
* Each learning unit MUST have its own unique ID that MUST be a unique, valid URL and MUST return an HTTP 200 OK response. The URL MUST NOT include '#' or any reserved URL query parameters used by the launcher to launch a specific learning unit as below.
* Each learning unit MUST have its own [Readium Web Publication Manifest](https://github.com/readium/webpub-manifest) example in [Appendix A](#appendix-a-sample-opds-catalogs) lesson001.json. 
* The manifest MUST be discoverable as per [Readium Web Publication Manifest Section 5](https://github.com/readium/webpub-manifest?tab=readme-ov-file#5-discovering-a-manifest) - this can be done using a link tag e.g. ```<link rel='manifest' type="application/webpub+json" href='lesson001.json'>``` or by adding a Link HTTP header e.g. ```Link: <https://example.app/id/lesson001>; rel="manifest"; type="application/webpub+json"``` as per the Readium spec.
* The [resources](https://github.com/readium/webpub-manifest?tab=readme-ov-file#21-sub-collections) section of the manifest MUST list all URLs required to run the Learning Resources (e.g. such that
those resources can be downloaded for later use offline, cached by network operators/schools, etc). All resource URLs and the URL (ID) for the learning unit itself:
  * MUST return an HTTP 200 response code in response to an HTTP GET request
  * MUST NOT use a vary header; the response content MUST be the same 
  * MUST include Content-Length header
  * MUST include a Last-Modified or ETag header
  * MUST support [cache validation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching#validation) using If-Modified-Since (when using Last-Modified header) or If-None-Match (when using ETag header)

**Launching a specific Learning Unit**

The RESPECT launcher app launches specific learning units using its URL ID deep link ([app links](https://developer.android.com/training/app-links) on Android); similar to how a deep link would be used to take the user from a messaging app to a specific destination in another app. 

The RESPECT launcher will add the following parameters to the learning unit URL (ID):

* respectLaunchVersion=1 Indicates that the launch is coming from a RESPECT Launcher app
* auth : The authentication to use with [xAPI](https://www.xapi.com) and/or [AGS](https://www.imsglobal.org/spec/lti-ags/v2p0/) API. 
* Some or none of the [OpenID Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), e.g. given_name, locale, sub,
  depending on the privacy settings being used by the RESPECT Launcher.
* endpoint_lti_ags : the HTTP url to use for the [LTI Assignment and Grade Services Specification](https://www.imsglobal.org/spec/lti-ags/v2p0/)
* endpoint : the HTTP url to use for xAPI ( named 'endpoint' because this is as per the Rustici launch spec, see below)
* All of the [Rustici Launch Parameters](https://github.com/RusticiSoftware/launch/blob/master/lms_lrs.md) as per the xAPI spec.

E.g. where a Learning Unit ID is https://example.org/topic/learningUnit1, then :
```
https://example.org/topic/learningUnit1/?respectLaunchVersion=1&auth=[secret]&given_name=John&locale=en-US
   &endpoint_lti_ags=http://localhost:8097/api/ags
   &endpoint=http://localhost:8097/api/xapi
   &actor={ "name" : ["Project Tin Can"], "mbox" : ["mailto:tincan@scorm.com"] }   
   &registration=760e3480-ba55-4991-94b0-01820dbd23a2   
   &activity_id=https://example.org/topic/learningUnit1/
```

When the respectLaunchVersion parameter is included, the client app MUST use the provided endpoints to send progress information back to the RESPECT
laucnher app and then finish (exit) the activity when the learning unit is complete.

**Expected outputs**
* OPDS listing learning units (as per OPDS spec above - see examples in [Appendix A](#appendix-a-sample-opds-catalogs))
* App built where:
  * Learning unit will open by launching the learning unit URL with the parameters as above (e.g. [deep link](https://developer.android.com/training/app-links/deep-linking))
  * Learner progress is sent to the xAPI and/or AGS server provided in the URL parameters. Any compliant server e.g. [ADL LRS](https://github.com/adlnet/ADL_LRS) or [LR SQL](https://github.com/yetanalytics/lrsql) can be used.
  * When the lesson is completed the app will finish (such that the user will return to the launcher).



# Appendix A: Sample OPDS catalogs

index.json (based on [OPDS specification example 2.1](https://drafts.opds.io/opds-2.0#21-navigation))
```
{
  "metadata": {
    "title": "Main Menu"
  },
  
  "links": [
    {"rel": "self", "href": "http://example.app/opds/index.json", "type": "application/opds+json"}
  ],
  
  "navigation": [
    {
      "href": "grade1.json", 
      "title": "Grade 1", 
      "type": "application/opds+json",
    },
    {
      "href": "grade2.json", 
      "title": "Grade 2", 
      "type": "application/opds+json", 
    }
  ]
}
```

**grade1.json** (based on [OPDS specification example 2.2](https://drafts.opds.io/opds-2.0#22-publications))
```
{
  "metadata": {
    "title": "Example listing publications"
  },
  
  "links": [
    {"rel": "self", "href": "http://example.app/opds/grade1.json", "type": "application/opds+json"}
  ],
  
  "publications": [
    {
      "metadata": {
        "@type": "http://schema.org/Game",
        "title": "Lesson 001",
        "author": "Mullah Nasruddin",
        "identifier": "https://example.app/id/lesson001",
        "language": "en",
        "modified": "2015-09-29T17:00:00Z",
        "subject": [
          {
            "name": "Mathematics",
            "scheme": "https://www.bisg.org/#bisac",
            "code": "MAT000000"
          }
        ]
      },
      "links": [
        {"rel": "self", "href": "http://example.app/opds/lesson001.json", "type": "application/opds-publication+json"},
        {"rel": "http://opds-spec.org/acquisition/open-access", "href": "http://example.app/lessons/lesson001/", "type": "text/html"}
      ],
      "images": [
        {"href": "http://example.org/cover-small.jpg", "type": "image/jpeg", "height": 700, "width": 400},
      ]
    },
    {
      "metadata": {
        "@type": "http://schema.org/Game",
        "title": "Lesson 002",
        "author": "Mullah Nasruddin",
        "identifier": "https://example.app/id/lesson002",
        "language": "en",
        "modified": "2015-09-29T17:00:00Z"
      },
      "links": [
        {"rel": "self", "href": "http://example.app/opds/lesson002.json", "type": "application/opds-publication+json"},
        {"rel": "http://opds-spec.org/acquisition/open-access", "href": "http://example.app/lessons/lesson002/", "type": "text/html"}
      ],
      "images": [
        {"href": "http://example.org/cover-small.jpg", "type": "image/jpeg", "height": 700, "width": 400},
      ]
    }   
  ]
}
```
Notes:
* Subjects use the BISAC or other schema as per [Readium Webpub Manifest spec](https://readium.org/webpub-manifest/contexts/default/#subjects)


lesson001.json (based on [OPDS specification example 5.1](https://drafts.opds.io/opds-2.0#51-opds-publication))
```
{
  "metadata": {
    "@type": "http://schema.org/Game",
    "title": "Lesson 001",
    "author": "Mullah Nasruddin",
    "identifier": "https://example.app/id/lesson001",
    "language": "en",
    "modified": "2015-09-29T17:00:00Z",
    
  },
  "links": [
    {"rel": "self", "href": "http://example.app/opds/lesson001.json", "type": "application/opds-publication+json"},
    {"rel": "http://opds-spec.org/acquisition/open-access", "href": "http://example.app/lessons/lesson001/", "type": "text/html"}
  ],
  "images": [
    {"href": "http://example.org/cover.jpg", "type": "image/jpeg", "height": 1400, "width": 800},
    {"href": "http://example.org/cover-small.jpg", "type": "image/jpeg", "height": 700, "width": 400},
    {"href": "http://example.org/cover.svg", "type": "image/svg+xml"}
  ],
  "resources": [
    {
      href: "http://example.app/lessons/lesson001/audio.ogg",
      type: "audio/ogg"
    },
    {
      href: "http://example.app/lessons/lesson001/video.mp4",
      type: "video/mp4"
    }
  ]
}

```
Notes:

* Resources list (see [Readium Web Publication Manifest Specification](https://github.com/readium/webpub-manifest?tab=readme-ov-file#21-sub-collections)) includes all resources required to use the learning unit
