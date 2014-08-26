---
layout: post
title: "Use WebIDE to debug Webapps on device over WIFI"
description: ""
category: Something you should know
tags: [mozilla, firefox os, WebIDE]
---
{% include JB/setup %}

WebIDE over WIFI is default disabled now, but you can enable it in four steps:

1. Turn on the developer menu: connect the phone to your PC via USB and execute `./edit-prefs.sh` under B2G folder. Append `user_pref("devtools.remote.wifi.visible", true);` to the end and save the file.

2. There will be a `DevTools via Wi-Fi` checkbox under Settings/Developer after reboot. Your phone can be discoverable after turn on this option.

<div style="text-align:center" markdown="1">
!["DevTools via Wi-Fi" in Developer menu]({{ site.url }}/assets/developer-menu.png)
</div>

3. Turn on the capability on Firefox Nightly: open `about:config` and change the `devtools.remote.wifi.scan` to true.

<div style="text-align:center" markdown="1">
![about:config on Firefox]({{ site.url }}/assets/about-config.png)
</div>

4. Open WebIDE and you'll see `WIFI Devices` in the `Select Runtime` menu. Your device should be appeared under this category if it's in the same local network.

<div style="text-align:center" markdown="1">
!["WIFI Devices" in WebIDE]({{ site.url }}/assets/webide.png)
</div>
