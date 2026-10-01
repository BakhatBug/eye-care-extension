# The Ultimate Guide to Chrome Extensions

Welcome! This guide will teach you the architecture and functioning of Chrome Extensions by walking through the real-world **Sleep Cycle Optimizer (`sleep-ext`)** project. By the end of this document, you'll understand how extensions are made, what files they require, and how the different components communicate.

---

## 1. Introduction: What is a Chrome Extension?

A Chrome Extension is essentially a small web application—built using HTML, CSS, and JavaScript—that modifies the browser's behavior or interacts with web pages. Unlike a standard website, an extension has access to special browser APIs (like alarms, storage, tabs, and notifications) that allow it to do much more than a regular web page.

---

## 2. Architecture Overview

Modern Chrome Extensions (using **Manifest V3**) are divided into several distinct components. Think of an extension as a team where each member has a specific job:

1. **The Core Config:** `manifest.json`
2. **The Brain (Background):** Service Workers (`background/worker.js`)
3. **The Hands (Content Scripts):** Code injected into web pages (`content/dimmer.js`, `content/dimmer.css`)
4. **The Face (UI Elements):** Popups and Options pages (`popup/popup.html`, `options/options.html`)

Let's dive into each component using your `sleep-ext` project as an example.

---

## 3. The Manifest (`manifest.json`)

The `manifest.json` file is the absolute most important file in any extension. It tells the browser everything it needs to know: what permissions to ask for, what scripts to run, what icon to show, and what the extension is called.

Here is the manifest from the Sleep Cycle Optimizer:

```json
{
  "manifest_version": 3,
  "name": "Sleep Cycle Optimizer",
  "version": "1.1.0",
  "permissions": [
    "tabs", "storage", "alarms", "scripting", "notifications", "idle"
  ],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background/worker.js"
  },
  "action": {
    "default_popup": "popup/popup.html",
    ...
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content/dimmer.js"],
      "css": ["content/dimmer.css"]
    }
  ],
  ...
}
```

### Key Takeaways:
- **`manifest_version: 3`**: Tells Chrome this is a modern extension.
- **`permissions`**: The extension requests access to specific browser features. E.g., `alarms` (to schedule bedtime alerts), `storage` (to save settings), `scripting` (to inject scripts).
- **`host_permissions`**: `["<all_urls>"]` means this extension can interact with *any* website you visit.
- **`background`**: Declares the background service worker.
- **`action`**: Declares what happens when you click the extension icon (it opens `popup.html`).
- **`content_scripts`**: Tells Chrome to automatically inject `dimmer.js` and `dimmer.css` into all websites (`<all_urls>`).

---

## 4. The Brain: Background Service Worker (`background/worker.js`)

The background script is a JavaScript file that runs invisibly in the background. In Manifest V3, this is called a **Service Worker**. 

**Its job is to manage the extension's lifecycle and react to browser events.** It does *not* have access to the webpage you are viewing (the DOM).

In `sleep-ext`, the `background/worker.js` handles things like:
- **Setting up Alarms**: Using `chrome.alarms.create()` to trigger a notification when it's your bedtime or time to take a break.
- **Listening to Events**: When an alarm fires (`chrome.alarms.onAlarm.addListener`), it might use `chrome.notifications.create()` to show a desktop alert.
- **Managing Global State**: Keeping track of whether "Eye Protection" is currently globally enabled or disabled.

> [!NOTE]
> Service workers are *ephemeral*. They wake up when an event happens (like an alarm firing or a message being received), do their job, and then go back to sleep. They do not run continuously.

---

## 5. The Hands: Content Scripts (`content/dimmer.js` & `css`)

While the background script controls the browser, **Content Scripts** control the actual web pages you visit. 

They are injected directly into the HTML of a webpage (like `google.com` or `youtube.com`). They *can* read and modify the DOM (the webpage content) but they *cannot* use most Chrome APIs.

In your project:
- **`content/dimmer.css`**: Contains styles for dimming the page and applying warm filters (for eye comfort).
- **`content/dimmer.js`**: Contains the logic to apply these styles. It listens for messages telling it to turn the dimmer ON or OFF, and then adds or removes specific CSS classes to the `<body>` of the webpage.

### How it works in `sleep-ext`:
Because `manifest.json` says:
```json
"matches": ["<all_urls>"],
"js": ["content/dimmer.js"]
```
Chrome automatically injects `dimmer.js` into every page you load. The script then waits for a signal to dim the screen.

---

## 6. The Face: User Interfaces (Popup & Options)

Extensions need a way for the user to interact with them.

### The Popup (`popup/popup.html` & `popup.js`)
This is the little window that appears when you click the extension icon in the toolbar.
- **HTML**: Provides the buttons and toggles (e.g., "Toggle Dimmer", "Toggle Eye Protection").
- **JS**: Listens for button clicks and sends messages to the background script or content script to make things happen.

### The Options Page (`options/options.html` & `options.js`)
This is a full-page settings menu where users can configure the extension.
- In `sleep-ext`, this is where users set their actual bedtime, break intervals, and dimming intensity. 
- The JS here uses `chrome.storage.sync.set()` to save these preferences so the background script can read them later.

---

## 7. Message Passing (The Nervous System)

Because Content Scripts, Background Scripts, and Popups all run in separate environments, they cannot share variables. They must communicate by sending messages to each other.

This is the most critical concept in extension development!

### Example Flow: Toggling the Dimmer
1. **User clicks "Toggle Dimmer" in the Popup.**
2. **Popup (`popup.js`) sends a message:**
   ```javascript
   chrome.runtime.sendMessage({ action: "toggleDimmer" });
   ```
3. **Background (`worker.js`) receives it:**
   The background script hears this, updates the storage (saying dimmer is ON), and then *broadcasts* a message to the active tab.
   ```javascript
   chrome.tabs.sendMessage(activeTab.id, { action: "dimmerStateChanged", state: true });
   ```
4. **Content Script (`dimmer.js`) receives it and acts:**
   The content script, sitting inside the webpage, hears the background script.
   ```javascript
   chrome.runtime.onMessage.addListener((message) => {
       if (message.action === "dimmerStateChanged") {
           // Apply the dark CSS filter to the webpage body!
           document.body.classList.add("sleep-ext-dimmed"); 
       }
   });
   ```

---

## 8. Summary of `sleep-ext` Architecture

Here is how your entire project comes together:

1. **Storage (`chrome.storage`)**: The central source of truth. The **Options page** writes settings here. The **Background worker** reads them to know when to fire alarms.
2. **Background (`worker.js`)**: Runs alarms and acts as a central router. When it's bedtime, it sends a message to the content scripts to dim the page.
3. **Content (`dimmer.js` & `dimmer.css`)**: Lives in every website. It blindly follows orders from the background script or popup, adding CSS overlays to protect your eyes.
4. **Popup (`popup.js`)**: A quick remote control. It reads the current state from storage and allows the user to manually override the background script (e.g., manually turning on the dimmer).

By mastering how these four pieces (Manifest, Background, Content Scripts, and UI) talk to each other via Message Passing, you can build any Chrome extension you can imagine!
