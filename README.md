## Chrome extension message passing made easy

### What?

Small library for sending messages across any extension parts (background, content script, popup or devtool).

It has a simple easy to use API, callback support and more.

### Why?

If you ever tried (like me) creating a fairly large extension which required communication between different parts, you might noticed that sending messages between the parts can get complicated and usually requires some relaying mechanism in the background page.
Adding callback functionality to these messages can make it even trickier.

Furthermore the chrome messaging API is not coherent or straight forward, sometimes requiring you to use _chrome.runtime.\*_ and sometimes _chrome.tabs.\*_ depending on which extension part you are currently in.

### How?
```javascript
npm install chrome-ext-messenger
```

#### 1) In the background page: Create a messenger instance and init the background hub.
```javascript
var Messenger = require('chrome-ext-messenger');
var messenger = new Messenger();

function connectedHandler(extPart, name, tabId) {
    console.log('someone connected:', arguments);
}

function disconnectedHandler(extPart, name, tabId) {
    console.log('someone disconnected:', arguments);
}

messenger.initBackgroundHub({
    connectedHandler: connectedHandler,
    disconnectedHandler: disconnectedHandler
});
```

This is obligatory for the library to work and should be done as early as possible in your background page.

If you are not using npm/es6, add the [library](https://github.com/asimen1/chrome-ext-messenger/tree/master/dist) via script tag and use _window['chrome-ext-messenger']_.

#### 2) Init connections (in any extension parts).
```javascript
// "name" - identifier name for this connection, can be any string except "*" (wildcard).
// "messageHandler" - handler for incoming messages to this connection.
messenger.initConnection(name, messageHandler)
```
For example:
```javascript
var Messenger = require('chrome-ext-messenger');
var messenger = new Messenger();

var messageHandler = function(message, from, sender, sendResponse) {
    if (message.text === 'HI!') {
        sendResponse('HOWDY!');
    }
};

var c = messenger.initConnection('main', messageHandler);
var c2 = messenger.initConnection('main2', messageHandler);

...
```

#### 3) Start sending messages across connections (in any extension parts).
```javascript
// "to" - where to send the message to: '<extension part>:<connection name>'.
//        <extension part> can be: 'background', 'content_script', 'popup', 'devtool'.
// "message" - the message to send.
// "responseCallback" - function that will be called if the receiver message handler invoked "sendResponse".
connection.sendMessage(to, message, responseCallback)
```
For example:
```javascript
// devtool.js -> content script
c.sendMessage('content_script:main', { text: 'HI!' }, function(response) {
   console.log(response);
});

// popup.js -> background
c.sendMessage('background:main', { text: 'HI!' }, function(response) {
   console.log(response);
});

// Messages from background to other parts require tab id: '<part>:<name>:<tabId>'.
// background.js -> content script ("150" is a tab id example).
c.sendMessage('content_script:main:150', { text: 'HI!' }, function(response) {
   console.log(response);
});

...
```

#### More:
```javascript
// Sending to multiple connections is supported using 'part:name1,name2,...'.
c.sendMessage('content_script:main,main2', { text: 'HI!' });

// Sending to all connections is supported using wildcard value '*'.
c.sendMessage('devtool:*', { text: 'HI!' });

// Disconnect the connection to stop listening for messages.
c.disconnect()
```

### Developing Locally
```javascript
// install dependencies
npm install
npm install webpack -g

// run the dev script
npm run dev
```
You can now use the built messenger from the _dist_ folder in a local test extension (or use [npm link](https://docs.npmjs.com/cli/link)).
I have created one (for internal testing purposes) that you can use: [chrome-ext-messenger-test](https://github.com/asimen1/chrome-ext-messenger-test).

### Notes
* Requires your extension to have ["tabs" permission](https://developer.chrome.com/extensions/declare_permissions).
* Uses only long lived port connections via _chrome.runtime.*_ API.
* This library should satisfy all your message passing demands, however if you are still handling some port connections manually using _chrome.runtime.onConnect_, you will also receive messenger ports connections. In order to identify connections originating from this library you can use the static method **Messenger.isMessengerPort(port)** which will return true/false.
* The Messenger messageHandler and _chrome.runtime.onMessage_ similarities and differences:
    * **Same** - "sender" object.
    * **Same** - "sendResponse" - The argument should be any JSON-ifiable object.
    * **Same** - "sendResponse" - With multiple message handler, the sendResponse() will work only for the first one to respond.  
    * **Different** - "from" object indicating the senders formatted identifier e.g. 'devtool:connection name'.
    * **Different** - Async sendResponse is supported directly (no need to return "true" value like with _chrome.runtime.onMessage_).

### Todos
* Support cross tabs communication (e.g. content script from tab 1 to content script of tab 2).
* connection.sendMessage: support array (multiples) in "toExtPart".
* connection.sendMessage: support * in "toTabIds" for background to non background (don't forget the "fromTabId" assignment...).

### Extensions using messenger
[Restyler](https://chrome.google.com/webstore/detail/restyler/ofkkcnbmhaodoaehikkibjanliaeffel)

License
----
MIT
