---
date: 2022-11-21
title: Discovering & reporting secrets bundled in an Electron app
---

> [!NOTE]
> To protect myself, the affected company & application names are changed.

## Background

In November 2022, as part of a consulting contract I found myself needing to configure Android devices to run in a form of kiosk-mode with
my contractor's line-of-business app as the sole application usable on the device.

The manufacturer of these Android devices, here-by referred to as BigCo, provided two features that allowed me to accomplish this at drastically reduced cost
compared to a traditional [mobile device management (MDM)](https://en.wikipedia.org/wiki/Mobile_device_management) solution:

1. BigCo provided an [Android Enterprise](https://www.android.com/enterprise/) configuration profile which pre-installed a special configuration agent app, here-by referred to as ConfigAgent, onto the devices.
2. BigCo provided a desktop application, here-by referred to as Configurator, which could be used to create configuration profiles for ConfigAgent.
   These profiles allowed you to do things like install apps onto the device, configure WIFI, configure kiosk-mode, etc.

Now, while Configurator worked, at the time BigCo only provided installers for Windows via their official download page.
My primary development environment is macOS, so this required me to boot a Windows VM whenever I wanted to make configuration changes-- which increasingly annoyed me.

## Investigation

While Configurator has a few advanced features such as the ability to run a local server for serving the ConfigAgent profiles,
its primary use-case is creating or updating the configuration profiles via a multi-step form.
The output of filling out the form is a Zip file containing an encrypted payload of the profile and a PDF with a QR code pointing to the publicly accesssible endpoint for the Zip file.

As Configurator has the ability to ingest these Zip files and pre-populate the form for updating profiles, my thinking was that if I can reverse engineer this encryption and decryption of the contents of the Zip file then I can create a small platform-agnostic CLI that converts JSON to the payload.
This would allows me to store configuration in plaintext in a secure version-controlled location and have a true CI/CD pipeline for pushing updates out to customers-- especially one that doesn't involve me or anyone else in the future booting a Windows VM.

Very quickly from interacting with Configurator I realized that it was a browser-based application.
They were using Material design and the particular lag of the animations on the drawer components smelled of an older Angular version to me.

Out of curiousity, I tried a right-click and surprising (or unsuprisingly 😅) was able to inspect element.
This was good news, I knew I could rip the JS payloads out the application via the Sources tab albeit in their minified/uglified form.

Before I went down that path though, I figured I might as well try extracting the application via [Asar](https://github.com/electron/asar).
Its highly likely this is an Electron app given development fads and then I could interact with everything via my editor:

```sh
7z x Configurator.exe

cd '$PLUGINSDIR/'
7z x app-64.7z

cd resources/
asar extract app.asar .
```

And just like that, I was in.

Unfortunately, navigating around I didn't see anything particularly helpful to my cause.
The logic that I cared about was likely hiding inside some ugly mess of `dist/main.js` so I figured the reverse-engineering path would take more time than it was worth.

While digging though, I saw two files that immediately set off some red flags: `id_rsa` and `.gitlab-ci.yml`.
A quick global search for `id_rsa` showed that it was used within `.gitlab-ci.yml`... for uploading builds to their public download server.
Additionally, right next to it in the same file there was a `GITHUB_TOKEN` for uploading builds to Github Releases.

My assumption is this is the result of a misconfiguration of [Electron-Builder](https://github.com/electron-userland/electron-builder) as their [default file exclusion filter](https://www.electron.build/configuration/contents) does not exclude either of these files.

## Resolution

After digging around on Github in hopes of finding a security contact, I ended up reaching out my contractor's Account Executive at BigCo.
Unfortunately, we ended up playing telephone for a few days as he relayed questions from some unknown engineer manager.
Finally, I decided to just forward the full reproduction details over email.

A few days, later I received this kind email & noticed that recent builds had been pulled from the various download locations:

![Bug Report Email](/posts/images/bug-report-email.png)

Those drinks never did happen, but happy to have made some friends over there 😂
