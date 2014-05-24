---
layout: post
title: "Analyzing an Android app with Docker and Androguard"
date: 2014-04-24 19:48:05 -0400
comments: true
favorite: true
author: David Weinstein
categories: docker androguard android analysis vulnerabilities
---

## TL;DR

Have you wanted to get into analyzing Android apps--for vulnerabilities,
malware, or just for fun? Maybe you've experienced problems getting an
environment setup because of complex tool dependencies, long READMEs, or just
never really knew where to get started? This article can help. Using docker I
can show you how to tear apart your first Android application, and maybe even
find some vulnerabilities while you're at it.

<!-- more -->

## Dependencies

I assume you have a working docker setup. If you aren't sure how to set it up
on your platform, try following one of these
[guides](http://docs.docker.io/installation/). 

## Download the repo

First check out the code from
[github](https://github.com/dweinstein/dockerfile-androguard) by running `git
clone git@github.com:dweinstein/dockerfile-androguard.git` or `git clone
https://github.com/dweinstein/dockerfile-androguard.git`.

This repo uses a `Dockerfile` to declare all the dependencies that you would
need if you were going to install the
[Androguard](https://code.google.com/p/androguard/) tool your self. If you want
to know how to do it all manually, you can
[see](https://code.google.com/p/androguard/wiki/Installation) for yourself.


## Build

After cloning the repository, `cd` into the directory and you should see something like this:

```
➜  dockerfile-androguard git:(master) ls
Dockerfile     Makefile       README.md      data-container
```

You can build by doing a `make build`

## Run

Once built, you have all you need to start analyzing your first app. I even put
an app in the repository for you to have something to play with right away! Just do a `make run`
and you should get something like this:

```
➜  dockerfile-androguard git:(master) make run
make -C data-container build
docker build -t androguard-data .
Uploading context 133.1 kB
Uploading context 
Step 0 : FROM busybox
 ---> 769b9341d937
Step 1 : ADD apks /apks
 ---> Using cache
 ---> 595c1e66e01f
Step 2 : VOLUME ["/apks"]
 ---> Using cache
 ---> 7d40787633a5
Step 3 : VOLUME ["/notes"]
 ---> Using cache
 ---> c0dc7db54a78
Step 4 : CMD ["/bin/true"]
 ---> Using cache
 ---> 845911656aad
Successfully built 845911656aad
make -C data-container run
docker run --name androguard_data androguard-data &> /dev/null
make[1]: [run] Error 1 (ignored)
docker run -t -i --rm --volumes-from androguard_data androguard
/usr/local/lib/python2.7/dist-packages/ipython-2.0.0-py2.7.egg/IPython/frontend.py:30: UserWarning: The top-level `frontend` package has been deprecated. All its subpackages have been moved to the top `IPython` level.
  warn("The top-level `frontend` package has been deprecated. "
Androlyze version 2.0
In [1]: 
```

This is an interactive shell session where you can begin to play with the apk
which I stored under `data-container/apks/test.apk`. Inside the container though it'll
show up as `/apks/test.apk`.


```
In [1]: ls -lh /apks/
total 128K
-rw-r--r-- 1 root root 125K Apr 23 15:10 test.apk
```

Load the APK into our APK object:

```
In [2]: apk = APK('apks/test.apk')
```

List the files inside of it:

```
In [3]: apk.get_files()
Out[3]: 
[u'META-INF/MANIFEST.MF',
 u'META-INF/APKIL.SF',
 u'META-INF/APKIL.RSA',
 u'res/layout/activity_main.xml',
 u'res/menu/activity_main.xml',
 u'AndroidManifest.xml',
 u'resources.arsc',
 u'res/drawable-hdpi/ic_action_search.png',
 u'res/drawable-hdpi/ic_launcher.png',
 u'res/drawable-ldpi/ic_launcher.png',
 u'res/drawable-mdpi/ic_action_search.png',
 u'res/drawable-mdpi/ic_launcher.png',
 u'res/drawable-xhdpi/ic_action_search.png',
 u'res/drawable-xhdpi/ic_launcher.png',
 u'classes.dex']
```

Load up some more complex analysis objects:

```
In [4]:  analysis = AnalyzeAPK('apks/test.apk')

In [5]: analysis
Out[5]: 
(<androguard.core.bytecodes.apk.APK instance at 0x1ce7c20>,
 <androguard.core.bytecodes.dvm.DalvikVMFormat at 0x1cdbfd0>,
 <androguard.core.analysis.analysis.uVMAnalysis instance at 0x326ed88>)
```

Get the application's [main
activity](http://developer.android.com/training/basics/activity-lifecycle/starting.html)
(the first class that runs when the app starts...

```
In [6]: apk.get_main_activity()
Out[6]: u'com.example.gangrene.GangreneActivity'

```

Get all classes/methods that have the word `gangrene` in them:
```
In [7]: show_Paths(analysis[1], analysis[2].tainted_packages.search_packages('gangrene'))
1 Lcom/example/gangrene/GangreneActivity;->onCreate(Landroid/os/Bundle;)V (0x34) ---> Lcom/example/gangrene/GangreneActivity$1;-><init>(Lcom/example/gangrene/GangreneActivity;)V
1 Lcom/example/gangrene/GangreneActivity$1;->onClick(Landroid/view/View;)V (0x1a) ---> Lcom/example/gangrene/GangreneActivity;->startService(Landroid/content/Intent;)Landroid/content/ComponentName;
1 Lcom/example/gangrene/GangreneActivity;->onCreate(Landroid/os/Bundle;)V (0xa) ---> Lcom/example/gangrene/GangreneActivity;->setContentView(I)V
1 Lcom/example/gangrene/GangreneActivity;->onCreate(Landroid/os/Bundle;)V (0x24) ---> Lcom/example/gangrene/GangreneActivity;->findViewById(I)Landroid/view/View;
1 Lcom/example/gangrene/GangreneActivity;->onCreateOptionsMenu(Landroid/view/Menu;)Z (0x0) ---> Lcom/example/gangrene/GangreneActivity;->getMenuInflater()Landroid/view/MenuInflater;
1 Lcom/example/gangrene/Feeder;->onCreate()V (0x54) ---> Lcom/example/gangrene/Feeder$ServiceHandler;-><init>(Lcom/example/gangrene/Feeder; Landroid/os/Looper;)V
1 Lcom/example/gangrene/Feeder;->onStartCommand(Landroid/content/Intent; I I)I (0x52) ---> Lcom/example/gangrene/Feeder$ServiceHandler;->obtainMessage()Landroid/os/Message;
1 Lcom/example/gangrene/Feeder;->onStartCommand(Landroid/content/Intent; I I)I (0x66) ---> Lcom/example/gangrene/Feeder$ServiceHandler;->sendMessage(Landroid/os/Message;)Z
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0x2c) ---> Lcom/example/gangrene/Feeder;->stopSelf(I)V
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0x6a) ---> Lcom/example/gangrene/Feeder;->getFilesDir()Ljava/io/File;
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0xb6) ---> Lcom/example/gangrene/Feeder;->access$0(Lcom/example/gangrene/Feeder;)I
```

What's this `Feeder` thing?

Let's find all "paths" for the package name `Feeder`. This results in us seeing all
classes/methods that have the word `Feeder` in their package path.

```
In [8]: show_Paths(analysis[1], analysis[2].tainted_packages.search_packages('Feeder'))
1 Lcom/example/gangrene/Feeder;->onCreate()V (0x54) ---> Lcom/example/gangrene/Feeder$ServiceHandler;-><init>(Lcom/example/gangrene/Feeder; Landroid/os/Looper;)V
1 Lcom/example/gangrene/Feeder;->onStartCommand(Landroid/content/Intent; I I)I (0x52) ---> Lcom/example/gangrene/Feeder$ServiceHandler;->obtainMessage()Landroid/os/Message;
1 Lcom/example/gangrene/Feeder;->onStartCommand(Landroid/content/Intent; I I)I (0x66) ---> Lcom/example/gangrene/Feeder$ServiceHandler;->sendMessage(Landroid/os/Message;)Z
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0x2c) ---> Lcom/example/gangrene/Feeder;->stopSelf(I)V
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0x6a) ---> Lcom/example/gangrene/Feeder;->getFilesDir()Ljava/io/File;
1 Lcom/example/gangrene/Feeder$ServiceHandler;->handleMessage(Landroid/os/Message;)V (0xb6) ---> Lcom/example/gangrene/Feeder;->access$0(Lcom/example/gangrene/Feeder;)I

```

Cool, so Feeder looks like an android
[Service](http://developer.android.com/reference/android/app/Service.html).
What happens when the `handleMessage` gets called back? Let's decompile the
whole apk (using [dad](https://code.google.com/p/androguard/wiki/Decompiler)),
but focus on the Feeder class for the moment:

```
In [9]: a, d, dx = AnalyzeAPK("apks/test.apk", decompiler="dad")
In [10]: print d.CLASS_Lcom_example_gangrene_Feeder.get_source()
package com.example.gangrene;
public class Feeder extends android.app.Service {
    final public java.util.Random randnumber;
    final private long durSec;
    private android.os.Looper mServiceLooper;
    private int bufferLengthBytes;
    private com.example.gangrene.Feeder$ServiceHandler mServiceHandler;
    final private int iterations;
    final public static String DTAG;
    final private int fileSizeBytes;

    public android.os.IBinder onBind(android.content.Intent p3)
    {
        android.util.Log.e("Feeder", "in onBind in Feeder service");
        return 0;
    }

    public void onCreate()
    {
        super.onCreate();
        android.util.Log.e("Feeder", "in onCreate in Feeder service");
        android.widget.Toast.makeText(this, "service created", 0).show();
        android.os.HandlerThread v0 = new android.os.HandlerThread("ServiceStartArguments", 10);
        v0.start();
        this.mServiceLooper = v0.getLooper();
        this.mServiceHandler = new com.example.gangrene.Feeder$ServiceHandler(this, this.mServiceLooper);
        return;
    }
...
```

That's pretty darn readable... because the APK was not
[obfuscated](http://developer.android.com/tools/help/proguard.html).

But going back to what was said earlier, we're interested in the `handleMessage` callback:

```
In [11]: print d.CLASS_Lcom_example_gangrene_Feeder_ServiceHandler.METHOD_handleMessage.source()
public void handleMessage(android.os.Message p24)
    {
        int v5 = 0;
        while(true) {
            int v4 = (v5 + 1);
            try {
                java.io.FileInputStream v14.close();
                v5 = v4;
            } catch (java.io.IOException v17) {
                v5 = v4;
            }
        }
        v5 = v4;
    }
```

That's odd, looks like an infinite loop that just closes an input stream (File)
over and over again... what a silly app. Looks like the author just wanted to
troll someone :-)

Here's what I think they meant to do:

```
int counter = 0;
while (counter++ < iterations) {

  long endTime = System.currentTimeMillis() + durSec * 1000;
  FileOutputStream fout = null;
  FileInputStream rand = null;
  int bytesRead = 0;
  long startTime = 0;
  float deltaTms = 0;

  try {
    fout = new FileOutputStream(getFilesDir() + "/test.out");
    rand = new FileInputStream("/dev/urandom");
    byte buffer[] = new byte[bufferLengthBytes];

    while (System.currentTimeMillis() < endTime
        && bytesRead < fileSizeBytes) {
      synchronized (this) {
        try {
          //Log.d(DTAG, String.format("in job loop %d, bytes: %d", counter, bytesRead));
          bytesRead += rand.read(buffer);
          // tic
          startTime = System.currentTimeMillis();
          fout.write(buffer);
          //fout.flush();
          deltaTms += (System.currentTimeMillis() - startTime);
          // wait(sleepSec*1000);
        } catch (IOException e) {
          Log.d(DTAG, "error reading/writing bytes");
        } catch (Exception e) {
          Log.e(DTAG, "other error while in job loop");
        }
      }

    }
    //toc


    if (deltaTms > 0) {
      Log.d(DTAG,
          String.format(
            "wrote %d bytes in %f ms; effective write speed %f Mbytes/sec",
            bytesRead,
            deltaTms,
            Float.valueOf(bytesRead / 1024 / 1024
              / (deltaTms / 1000))));
    }
    ...
```

## Conclusion

More people should use docker to deploy tools to their community. It would make
everyone's life easier and less time would be spent dealing with dependencies
and more time getting stuff done.

Happy Hacking!


