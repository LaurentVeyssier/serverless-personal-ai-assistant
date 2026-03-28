![assistant](serverless-assistant.png)


# 🚀 Project: The "Zero-Cost" Personal AI Agent

**An automated pipeline to capture, analyze, and archive information from your iPhone to Google Sheets using Gemini 2.5 flash**

---

## 📖 Overview

This project turns your iPhone's **Share Sheet** into a powerful data entry tool.

* **The Trigger:** Share any URL or text from any app.
* **The Brain:** Google Gemini 2.5 Flash (via API) analyzes the content based on a specific "Task ID."
* **The Database:** A Google Sheet that automatically categorizes and deduplicates your entries.

---

## 🛠️ The "Serverless" Tech Stack

* **Mobile Interface:** iOS Shortcuts (Native)
* **Logic Engine:** Google Apps Script (JavaScript / Web App)
* **AI Model:** Google Gemini 2.5 Flash (Free Tier)
* **Storage:** Google Sheets

---

## 📜 The "Universal Router" Code

Copy this into your **Google Apps Script** editor (**Extensions > Apps Script** from your sheet).
Save and hit Deploy as a web app ("Execute as me" + "who has acces: anyone" options).
Copy the app URL provided once deployed. The URL points to your google sheet and will be used to link your iphone shortcut to the google sheet.
Extracted information will be saved in a google sheet tab named after the "sheetName" defined in getTaskConfiguration function (eg "AI Research").

```javascript
// --- CONFIGURATION ---
const GEMINI_API_KEY = 'YOUR_API_KEY_HERE';
const GEMINI_URL = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}`;

/**
 * MAIN ENTRY POINT: Receives data from iPhone
 */
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const rawInput = data.message; 
  const taskType = data.task || "AI_NEWS"; 

  // 1. Get the Task-Specific Logic
  const taskConfig = getTaskConfiguration(taskType);

  // 2. Perform AI Analysis
  const analysis = callGemini(rawInput, taskConfig.prompt);

  // 3. Persist with Deduplication
  const status = saveToSheet(analysis, taskConfig.sheetName);

  return ContentService.createTextOutput(JSON.stringify({"status": status, "data": analysis}))
    .setMimeType(ContentService.MimeType.JSON);
}

/**
 * THE ROUTER: Define your apps/prompts here
 */
function getTaskConfiguration(type) {
  const tasks = {
    "AI_NEWS": {
      sheetName: "AI Research",
      prompt: "Analyze this URL. Extract: Title, Summary (2 sentences), and Technical Impact (Low/Med/High). Return ONLY JSON: {title, summary, impact}."
    },
    "EXPENSES": {
      sheetName: "Finances",
      prompt: "Extract: Merchant, Total Amount, and Category. Return ONLY JSON: {merchant, amount, category}."
    }
  };
  return tasks[type] || tasks["AI_NEWS"];
}

/**
 * PERSISTENCE: Save & Deduplicate
 */
function saveToSheet(data, sheetName) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName) || ss.insertSheet(sheetName);
  
  if (sheet.getLastRow() === 0) sheet.appendRow(Object.keys(data));

  const existingData = sheet.getRange(1, 1, sheet.getLastRow() || 1, 1).getValues().flat();
  if (existingData.includes(Object.values(data)[0])) return "Duplicate Skipped";

  sheet.appendRow(Object.values(data));
  return "Success";
}

/**
 * BRAIN: API Call to Gemini
 */
function callGemini(input, systemPrompt) {
  const payload = { "contents": [{ "parts": [{ "text": `${systemPrompt}\n\nInput: ${input}` }] }] };
  const response = UrlFetchApp.fetch(GEMINI_URL, {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify(payload)
  });
  const cleanText = JSON.parse(response.getContentText()).candidates[0].content.parts[0].text.replace(/```json|```/g, "").trim();
  return JSON.parse(cleanText);
}

```

---

## 📲 How to Setup the iPhone Shortcut

Open your Shortcut app on your iphone then follow these steps:

1. **New Shortcut:** Name it "Send to AI."
2. **Settings:** Enable **"Show in Share Sheet."**
3. Search "Dictionary" and add to the workflow
4. Add new element as Text
* Define key:text as `message` ➔ `Shortcut Input`
* `task` ➔ `AI_NEWS`
5. Search for "URL" and select "Get URL content"
6. In URL filed, paste your google sheet URL endpoint
* Set Method to **POST**.
* Set Request Body to **File** ➔ Select the **Dictionary** created above.
7. **Run:** Share any link from your phone using "Send to AI" in your menu options

---

## ✨ Why this Architecture?

* **Modular:** To add a new feature (e.g., "Business Profile"), you just add one line to the `tasks` object in the script. You don't need to rebuild the infrastructure.
* **Context-Aware:** The script automatically visits URLs or parses raw text depending on what you share.
* **Scalable:** Handles deduplication so you don't clutter your database with the same news twice.
