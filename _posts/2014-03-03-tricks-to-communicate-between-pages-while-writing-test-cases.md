---
layout: post
title: "Tricks to communicate between pages while writing test cases"
description: ""
category: "Something you should know"
tags: [mozilla, gecko, mochitest]
---
{% include JB/setup %}

Here are some techniques I've used while writing Gecko test cases. It's quite handy for people to create complicate test cases across multiple frames or processes.

To non-OOP IFrame
---------
[postMessage][1]: You can pass the message between iframes that not in the same origin, using `postMessage` and `addEventListener('message', callback)`.

outer.html

    var iframe = document.getElementById('iframe');
    iframe.src = 'inner.html';

    window.addEventListener('message', function(evt) { /* handle the message from inner.html */ });
    iframe.onload = function() { iframe.contentWindow.postMessage('some-js-value', window.location.origin) };

inner.html

    window.addEventListener('message', function(evt) {
      // handle the message from outer.html
      if (evt.data === 'some-js-value') {
        event.source.postMessage('something-else', window.location.origin);
      }
    }

[URL Query String][2]: You can pass the information in URL while opening a page in iframe, using [`encodeURIComponent`][3] and [`decodeURIComponent`][4].

outer.html

    var iframe = document.getElementById('iframe');
    iframe.src = 'inner.html?' + encodeURIComponent('some string or json string');

inner.html

    var queryString = decodeURIComponent(window.location.search.substring(1));
    // use JSON.parse(queryString) if passing a json string

To Parent Process
-----------------
[loadChromeScript][5]: You can load a specified js file into B2G process and you can use `sendAsyncMessage` to pass information between chrome process and content process.

test.html

    var script = SpecialPowers.loadChromeScript(SimpleTest.getTestFileURL('chrome-script.js');
    script.addMessageListener('some-command', function(data) {
      // handle the message from chrome-script.js
    });
    script.sendAsyncMessage('some-message', 'some js value');
    script.destroy();

chrome-script.js

    addMessageHandler('some-message', function(data) {
      // handle the message from test.html
      sendAsyncMessage('some-command', 'some other js value');
    });

[window.alert][6]/[window.prompt][7]: If you create a content process in your testcase, you can use `window.alert` or `window.prompt` to initiate a IPC from content process to parent process by handling [`mozbrowsershowmodalprompt`][8] event.

outer.html

    function callback() {
      var iframe = document.createElement('iframe');
      SpecialPowers.wrap(iframe).mozbrowser = true;

      iframe.addEventListener('mozbrowsershowmodalprompt', function(e) {
        // handle the message from inner.html
        var message = e.detail.message;
        var initValue = e.detail.initValue; // init value from prompt.
        e.detail.returenValue = 'some string'; // return value for prompt.

        e.preventDefault(); // cause the alert to block.
        e.detail.unblock(); // Now unblock the iframe and check that the script completed.

      });

      iframe.src = 'inner.html';
    }

    // necessary permission and preference for creating a remote iframe
    SpecialPowers.addPermission("browser", true, document);
    SpecialPowers.pushPrefEnv({
      "set": [
        ["network.disable.ipc.security", true],
        ["dom.ipc.tabs.disabled", false],
        ["dom.ipc.browser_frames.oop_by_default", true],
        ["dom.mozBrowserFramesEnabled", true],
        ["browser.pagethumbnails.capturing_disabled", true]
      ]
    }, callback);

inner.html

    window.alert('some string or json string');
    var retValue = window.prompt('some thing need return', 'some init value');

To Child Process
----------------
[hashchange][9]: You can pass the information by changing the URL hash for the iframe, using [`encodeURIComponent`][3] and [`decodeURIComponent`][4].

outer.html

    var iframe = document.getElementById('iframe');
    iframe.src = 'inner.html#' + encodeURIComponent('some string or json string');

inner.html

    window.addEventListener('hashchange', function() {
      var queryString = decodeURIComponent(window.location.hash.substring(1));
      // use JSON.parse(queryString) if passing a json string
    });

[1]: https://developer.mozilla.org/en-US/docs/Web/API/Window.postMessage
[2]: http://en.wikipedia.org/wiki/Query_string
[3]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
[4]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/decodeURIComponent
[5]: https://developer.mozilla.org/en-US/docs/SpecialPowers#loadChromeScript()
[6]: https://developer.mozilla.org/en-US/docs/Web/API/window.alert
[7]: https://developer.mozilla.org/en-US/docs/Web/API/window.prompt
[8]: https://developer.mozilla.org/en-US/docs/Web/Reference/Events/mozbrowsershowmodalprompt
[9]: https://developer.mozilla.org/en-US/docs/Web/API/Window.onhashchange
