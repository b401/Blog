---
author: i4
title: "Finding a new apartment with fast internet"
date: M01-15-2021
---

## Hello friends!
Some time ago I was looking for a new place to stay with reasonably fast internet. I used [Comparis](https://www.comparis.ch/immobilien/default) as my goto platform.
After I started to look at the various locations I became tired of always copy/pasting the location into our ISPs website to check the supported speed for the location. 

There needs to be another way!

## TLDR;
Wrote an extension for Chromium based browsers (Chrome,Brave,Edgeium) to inject additional vital informations into the searchresults of comparis. 

You can download and look at it here: [Github link](https://github.com/b401/Comparis-fiber-checke)

Currently it's in review on the Chromestore. So please if you want to check it out use the repository files on my master branch.

## Writing a browser Add-On
A browser add-on (or extension) is nothing more than (mostly) a set of javascript files that can interact with the browser and it's content.

In my case the extension had the following dirtree:

- _images_
-- _xxx.png_
- _background.js_
- _contentscript.js_
- _manifest.json_

### manifest.json
The manifest is used to inform the browser about the needed permissions and basic informations.

If you're interested in how the whole manifest looks like - see manifest.json on my github.

I started to look at how a browser add-in can work with the [DOM](https://www.w3schools.com/js/js_htmldom.asp) and inject new content into it.
This isn't really hard as the add-on only needs permission on the tab and sites it will interact with. To do this which will be defined in the manifest.json.

```json
"permissions":[
	"https://fiber.salt.ch/",
	"https://api.init7.net/",
	"https://www.swisscom.ch/",
	"activeTab"
]
```

### popup.html
This is the popup that a user gets to see if he clicks on you extension. As I'm not showing the user any information, this will just be a html skel page.

```html
<!DOCTYPE html>
<html>
  <head>
    <style>
    </style>
  </head>
  <body>
  </body>
</html>
```

The background.js gets later dynamically injected.

### background.js
The backgroundscript.js includes functions for "API" calls to the external ISP websites. It's a little bit messy and needs to be rewrote into classes. The only thing of interest here, are the following lines:

```javascript
// Use chrome API to listen for new requests.
// This is needed because of CORB function which Google introduced to fight
// against XSS and other vulnerabilities.
chrome.extension.onRequest.addListener(function(message, sender, sendResponse) {
    if (message.title === 'showResponse') {
        wrapper(message.homes, sendResponse, function(values) {
            sendResponse(values);
        });
    };
});
```

So yeah... What the fuck is CORB? 
> Origin Read Blocking (CORB) is a new web platform security feature that helps mitigate the threat of side-channel attacks (including Spectre).  It is designed to prevent the browser from delivering certain cross-origin network responses to a web page, when they might contain sensitive information and are not needed for existing web features.  For example, it will block a cross-origin text/html response requested from a \<script\> or \<img\> tag, replacing it with an empty response instead.

Basically the main problem is that we can't send _fetch()_ requests directly from our backround.js. To circumvent this, I use a wrapper for the calls and send the request from an additional script in the contenscript.js and use a listener to wrap them up.

More on: [CORB explained](https://chromium.googlesource.com/chromium/src/+/master/services/network/cross_origin_read_blocking_explainer.md#The-problem).

### contentscript.js
I use an _MutationObserver_ to check for new added entries as comparis injects the new apartments as divs dynamically.
```javascript
var observer = new WebKitMutationObserver(function(mutations) {
    mutations.forEach(function(mutation) {
        if (mutation.addedNodes.length > 0) {
            if (mutation.addedNodes[0].tagName === 'DIV') {
                var xpath = document.evaluate(".//*[local-name() = 'svg' and @data-icon='map-marker-alt']", mutation.addedNodes[0], null, XPathResult.FIRST_ORDERED_NODE_TYPE, null);
                if (xpath.singleNodeValue !== null && xpath.singleNodeValue !== undefined) {
                    main(xpath.singleNodeValue);
                }
            }
        }
    });
});

observer.observe(document, {
    childList: true,
    subtree: true
});
```

I am only interested in svg elements as I travers my way up to find the right element with the location informations.

In the rest of the code I just make sure that we get the needed data from the DOM.
```javascript
let id = newobject.parentElement.parentElement;
let adress = id.innerText;
let zip_code = adress.match(/[\d]{4}\s/i)[0].trim();
let city = adress.match(/[\d]{4}\s.+/i)[0].replace(zip_code, "").trim();
let street_number = adress.replace(zip_code, "").replace(city, "").replace(",", "").trim();
```

If there's no street_number we can't get informations out of the ISP APIs, so I just drop it.
Yeah I also don't like regex but it works :)


## Why should you use it?
### Without Comparis-Fiber-Checker
![without extensions](/b/images/withouth_extension.png)

### With Comparis-Fiber-Checker
![without extensions](/b/images/with_extension.png)

## Conclusion
As Corona is still rampant it's important to know if the internet of your new home will be fast enough to support your homeoffice efforts. For myself it's one of the top priorities.
