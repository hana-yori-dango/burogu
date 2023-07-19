+++
title = "KanjiSensei"
weight = 1
order = 1
date = 2023-07-19
insert_anchor_links = "right"

[taxonomies]
tags = ["Web extension", "Javascript", "HTML", "CSS"]
+++

[KanjiSensei](https://chrome.google.com/webstore/detail/kanjisensei-learn-kanji-w/bnfleamolloogjkddoaocllogcokncfl) is a Chrome web extension to learn Kanji while browsing the web.

<!-- more -->

![dark](https://cluzeau.pro/images/2023-07-19-kanji-sensei/dark.png)

## TLDR

While browsing, the extension will replace English or French words found in `<p>` tags with their Kanji equivalent.

For the first release, only Kanji in JLPT N5 are available.

While hovering the kanji, a tooltip is displayed with the following information:

| Field | Example |
| --- | --- |
| the replaced word in the web page | Moon
| the number of strokes to write the kanji | 4 |
| the JLPT level | N5 |
| the kanji | 月 |
| the meaning of the kanji | month, moon |
| the [on-yomi](https://en.wikipedia.org/wiki/Kanji#On'yomi_(Sino-Japanese_reading)) (romaji & katakana) | GETSU, GATSU ゲツ,ガツ|
| the [kun-yomi](https://en.wikipedia.org/wiki/Kanji#Kun'yomi_(native_reading)) (romaji & hiragana) | tsuki つき |

## How it works

### Disclaimer

This was my first web extension, so I am sure there are better ways to do this. :-)

There should not be many mistakes (since it works), though it has been simplified for the sake of this article.

### Project layout

```
src
├── content.js     # web page updating logic
├── dictionary.js  # function findKanji(word, language)
├── tooltip.js     # class extending HTMLElement to display the tooltip
├── tooltip.css
│
├── popup.html     # settings page, opened when clicking on the extension icon
├── popup.js       # popup logic that update the chrome store on user actions
├── popup.css
├── icon.png       # icon in the top right of your browser
│
└── manifest.json  # browser extension manifest, required to make it all work together
```

### Content script

```js
function modifyParagraphs(state) {
  // modify paragraphs based on the settings and the page language
}


// runs on page load
chrome.storage.local.get([ 'toggleState', ... ], function(localState) {
  const toggleState = localState.toggleState || 'off';
  // [...]

  if (toggleState === 'on') {
    // when loading the page, if the extension is on, we modify the paragraphs
    modifyParagraphs({ ... });
  }
});

// runs when the extension popup sends a message
chrome.runtime.onMessage.addListener(function(message, sender, sendResponse) {
  // [...]

  if (event == 'toggle-clicked') {
    // reloading the page to remove our modifications or to modify the page
    location.reload();
  }
});
```

### Popup

![popup](https://cluzeau.pro/images/2023-07-19-kanji-sensei/popup.png)


Example of `popup.html` & `popup.js` with a `on/off` toggle button.

Here we have a `toggleButton` that is checked or not depending on the `toggleState` stored in `chrome.storage.local`.

When the `toggleButton` is clicked, we update the `toggleState` and send a message to the content script to update the `toggleState` there as well and enable or disable word replacement.

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="stylesheet" href="popup.css" />
  </head>
  <body>
    <div>
      <!-- [...] -->
      <input type="checkbox" id="toggleExtension" />
      <!-- [...] -->
    </div>
    <!-- the script needs to be loaded after the html body -->
    <script src="popup.js"></script>
  </body>
</html>
```

```js
document.addEventListener('DOMContentLoaded', function() {
  // we get a reference to the toggle button  
  const toggleButton = document.getElementById('toggleExtension');

  // we get the toggle state from the storage & update its value from the user interaction
  chrome.storage.local.get(['toggleState'], function(data) {
    let toggleState = data.toggleState || 'off';
    toggleButton.checked = toggleState === 'on';

    toggleButton.addEventListener('click', () => {
      toggleState = toggleState === 'on' ? 'off' : 'on';
      toggleButton.checked = toggleState === 'on';

      // we store the toggle state on every click the user makes on the toggle
      chrome.storage.local.set({toggleState});

      // we send a message to the content script to update its state accordingly and update the page
      updateContentScript('toggle-clicked');
    });

    function updateContentScript(event) {
      chrome.tabs.query({active: true, currentWindow: true}, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, { toggleState });
      });
    }

  });
});
```

### Manifest version 3

Make sure to only use required permissions otherwise the extension will be rejected when submitted to the chrome store.

To avoid conflicts with the web page css, we have specific `css` for the popup and the tooltip and also custom classes.

To ensure our extension can run on any webpage, we match on `<all_urls>`.

```json
{
  "manifest_version": 3,
  "name": "KanjiSensei",
  "permissions": ["activeTab", "storage"],
  "action": {
    "default_popup": "popup.html"
  },
  "icons": {
    "48": "icon.png"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content.js"],
      "run_at": "document_idle",
      "css": ["tooltip.css"]
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["popup.css"],
      "matches": ["<all_urls>"]
    }
  ]
}
```

## Links

- [https://developer.chrome.com/docs/extensions/](https://developer.chrome.com/docs/extensions/)
- [https://developer.mozilla.org/.../WebExtensions](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions)
