---
title: Frida and time-based logout
description: Bypassing application logout with frida
date: 2022-10-12T14:33:01.394294+02:00
---

# Introduction

Last couple of days, I had to do the penetration test of an iOS application. I first had to deal with the bypassing encryption, turned out it used just good old `CCCrypt` for which I have created a small [script](https://codeshare.frida.re/@lateralusd/cccrypt/).

While analyzing the application, I have noticed that every couple of seconds I get logged out. Because I was annoyed by it, I have decided to bypass it.

## Analyzing and bypassing

The first thing that happens during the logout is that `UIAlertController` is presented with the message _"Your session has expired. Please log in again!"_.

Steps to create UIAlertController are:

* Create `UIAlertController` 
* Create `UIAlertAction` 
* Call `-[UIAlertController addAction:]`

The idea was to intercept `-[UIAlertController addAction:]` and once I am there, I will print the backtrace. The frida script to intercept the method and show the stacktrace is shown below.

```js
var addAction = ObjC.classes.UIAlertController["- addAction:"].implementation;

Interceptor.attach(addAction, {
	onEnter: function(args) {
		console.log("Showing alert");
		console.log("Got called from\n" +
			Thread.backtrace(this.context, Backtracer.ACCURATE)
        			.map(DebugSymbol.fromAddress).join('\n') + '\n');
	},
	onLeave: function(args) {

	}
})
```

After a couple of minutes, backtrace is shown inside the frida:

```bash
$ frida -U "Test APP" -l /tmp/script.js
[iPhone::Test App ]-> Showing alert
Got called from
0x18f8352f4 UIKitCore!-[UIAlertView _prepareAlertActions]
0x18f8359ac UIKitCore!-[UIAlertView _setIsPresented:]
0x18f836274 UIKitCore!-[UIAlertView _showAnimated:]
0x10250b8a8 /private/var/containers/Bundle/Application/XXXXXXXXXXXXXXXXXXXXXXXXX/Test App.app/Test App!-[AppDelegate showMessage:messageType:]
0x10250be8c /private/var/containers/Bundle/Application/XXXXXXXXXXXXXXXXXXXXXXXXX/Test App.app/Test App!-[AppDelegate idleTimerExceeded]
0x18c598030 Foundation!__NSFireTimer
0x18c12d03c CoreFoundation!__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
0x18c12cd78 CoreFoundation!__CFRunLoopDoTimer
0x18c12c448 CoreFoundation!__CFRunLoopDoTimers
0x18c127584 CoreFoundation!__CFRunLoopRun
0x18c126adc CoreFoundation!CFRunLoopRunSpecific
0x1960ac328 GraphicsServices!GSEventRunModal
0x190221ae0 UIKitCore!UIApplicationMain
0x102511cc4 Test App!0x2dcc4 (0x10002dcc4)
0x18bfb0360 libdyld.dylib!start


```

We can notice that `-[AppDelegate idleTimerExceeded]` calls `-[AppDelegate showMessage:messageType:]` which in turns shows us our alert.

Loading the app in the Hopper and inspecting the `- idleTimerExceeded` showed us that the application does the following:

```
 r0 = self;
    var_20 = r22;
    stack[-40] = r21;
    r31 = r31 + 0xffffffffffffffd0;
    var_10 = r20;
    stack[-24] = r19;
    saved_fp = r29;
    stack[-8] = r30;
    r29 = &saved_fp;
    if (*(int8_t *)(int64_t *)&r0->logedIn != 0x0) {
            r19 = r0;
            if (*(int8_t *)(int64_t *)&r0->connecting == 0x0) {
                    r0 = [UIApplication sharedApplication];
                    r0 = [r0 retain];
                    r21 = [[r0 windows] retain];
                    [r19 checkViews:r21];
                    [r21 release];
                    [r0 release];
                    [r19 logedOut:r19 withAnimation:0x0];
                    [[Language get:@"AppDelegate.label.inactivityError"] retain];
                    [r19 showMessage:r2 messageType:r3];
                    [r20 release];
            }
    }
    return;
```

Flow: 

* Prepare the basic variable
* Check if the user is logged in
* If the user is logged in, log out the user and show the alert
* If the user is not logged in, just return

So, in order to bypass this, we just needs to set `logedIn` to 0x1(false) and we have successfully bypassed the check. Final script and the run are shown below:

```
$ cat /tmp/bypass.js
var idleTimerExceeded = ObjC.classes.AppDelegate["- idleTimerExceeded"].implementation;

Interceptor.attach(idleTimerExceeded, {
        onEnter: function(args) {
                console.log("[*] Inside the idleTimerExceeded method");
                var appDelegate = ObjC.Object(args[0]); // Create object from args[0] -> AppDelegate
                appDelegate.$ivars.logedIn = false; // Set the logedIn variable to false
        },
        onLeave: function(retval) {

        }
})
$ frida -U "Test App" -l /tmp/bypass.js
[iPhone::Test App ]-> [*] Inside the idleTimerExceeded method
[iPhone::Test App ]->
[iPhone::Test App ]-> var delegate = ObjC.classes.UIApplication.sharedApplication().delegate();
[iPhone::Test App ]-> delegate.$ivars.logedIn
false
[iPhone::Test App ]->

```

After the method got called, we have set `logedIn` to false in order to bypass the check and this way I wont be logged out again every couple of minutes.