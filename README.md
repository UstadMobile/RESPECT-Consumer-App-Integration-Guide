# RESPECT Consumer App Integration Guide

Terms:

__Learning Unit__: A distinct learning unit that can be assigned to a learner. This may or may not involve any assessment. The RESPECT launcher app can be used to launch any RESPECT Compatible app itself as well as launch any specific learning unit.

## Steps

1) Create a JSON manifest and publish it on any https server.
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


