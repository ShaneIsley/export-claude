# Claude Exporter

A bookmarklet to extract and download your Claude data. You can download individual projects or all of your projects at once, either as raw JSON or as a neatly formatted ZIP file full of Markdown documents!

## Features (v1.0.0)
* **Interactive UI:** Choose exactly what to export directly inside Claude.
* **ZIP Export:** Converts your chats to beautiful Markdown files, saves your uploaded knowledge files, and extracts code artifacts automatically.
* **JSON Export:** Download your data as raw JSON if you want to write scripts or re-import it later.
* **Global Export:** Option to download all of your Projects and orphaned chats in one click.

## How to Install

There are two ways to install: one will run the latest code dynamically, the other copies the current code into a static bookmark(let). Go with what is most comfortable. 

**1. Copy the tiny code block below:**
```javascript
javascript:(function(){var s=document.createElement('script');s.src='https://cdn.jsdelivr.net/gh/ShaneIsley/export-claude/export-claude.js';document.body.appendChild(s);})();
```

**2. Create a new Bookmark:**
* Bookmark this page (or any random webpage).
* Make sure your Bookmarks Bar is visible (`Ctrl+Shift+B` on Windows/Linux or `Cmd+Shift+B` on Mac).

**3. Edit the Bookmark:**
* Right-click the newly created bookmark in your Bookmarks Bar and select **Edit**.
* Change the **Name** to `Claude Export` (or whatever you prefer).
* Delete the website address in the **URL** box, and **Paste** the code you copied in Step 1.
* Click **Save**.

*(Note: Because this script loads dynamically from GitHub, you will automatically get changes in the future without having to reinstall the bookmarklet)*

---

## Manual Installation

Make a new bookmark and paste the code of [export-claude.js] (https://raw.githubusercontent.com/ShaneIsley/export-claude/main/export-claude.js) into the URL box. Now click the bookmark when on the claude site to run an export.

---

## How to Use
1. Log into [Claude.ai](https://claude.ai/).
2. Click the `Claude Export` bookmarklet in your Bookmarks Bar.
3. A menu will appear. Choose whether you want to export your Current Project or All Projects, and choose between a ZIP file or a JSON file.
4. Wait for the extraction to complete. Your file will automatically download when it finishes!
