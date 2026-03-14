# export-claude: A Code Walkthrough

*2026-03-14T22:56:48Z by Showboat 0.6.1*
<!-- showboat-id: aab1690e-0001-4c6c-bc90-2a14b9cd5f0b -->

## Overview

export-claude is a browser bookmarklet that exports your Claude.ai conversations and project data. When you click the bookmarklet, it runs directly in your browser tab, uses your existing login session to call Claude.ai's private API, and downloads everything as either a structured JSON file or a ZIP archive of Markdown files with code artifacts extracted and organized into folders.

The entire application lives in two files:
- `export-claude.js` — minified production version (loaded by the bookmarklet)
- `export-claude-max.js` — the same code formatted for readability (what we walk through here)

Let's start with a high-level look at the file.

```bash
wc -l export-claude-max.js && echo '---' && head -2 export-claude-max.js && echo '...' && tail -2 export-claude-max.js
```

```output
680 export-claude-max.js
---
(function() {
  'use strict';
...
  });
})();
```

## 1. The Outer Wrapper (lines 1 & 680)

The entire file is wrapped in an **IIFE** (Immediately Invoked Function Expression). This is the standard browser bookmarklet pattern: the function runs immediately when the script loads and keeps all variables out of the global `window` scope, preventing conflicts with Claude.ai's own JavaScript. The `'use strict'` directive enables stricter error checking.

```bash
sed -n '1,11p' export-claude-max.js
```

```output
(function() {
  'use strict';

  var CONFIG = {
    VERSION: '2.1.0',
    API_BASE: '/api',
    PAGE_SIZE: 50,
    CHAT_DELAY_MS: 750,
    PROJECT_DELAY_MS: 1000,
    JSZIP_CDN: 'https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js'
  };
```

## 2. CONFIG (lines 4–11)

A single global constants object. Key values to note:
- **`PAGE_SIZE: 50`** — how many items to request per paginated API call
- **`CHAT_DELAY_MS: 750`** — 750ms pause between fetching individual chat conversations (rate-limit courtesy)
- **`PROJECT_DELAY_MS: 1000`** — 1s pause between processing each project during an 'export all'
- **`JSZIP_CDN`** — JSZip is an external dependency loaded on demand from a CDN only when the user selects ZIP format

## 3. Utils (lines 14–38)

Five small helper functions used throughout the codebase.

```bash
sed -n '14,38p' export-claude-max.js
```

```output
  var Utils = {
    getOrgId: function() {
      var c = document.cookie.match(/lastActiveOrg=([^;]+)/);
      if (c) return c[1];
      var entries = performance.getEntriesByType('resource');
      for (var i = 0; i < entries.length; i++) {
        var m = entries[i].name.match(/\/api\/organizations\/([a-f0-9-]+)/);
        if (m) return m[1];
      }
      return null;
    },
    getProjectId: function() {
      var m = window.location.pathname.match(/\/project\/([a-f0-9-]+)/);
      return m ? m[1] : null;
    },
    timestamp: function() {
      return new Date().toISOString().slice(0, 19).replace(/[:.]/g, '-');
    },
    sleep: function(ms) {
      return new Promise(function(r) { setTimeout(r, ms); });
    },
    safeName: function(n) {
      return (n || 'untitled').replace(/[^a-zA-Z0-9_\- ]/g, '').replace(/\s+/g, '-').replace(/-{2,}/g, '-').replace(/^-|-$/g, '').slice(0, 60) || 'untitled';
    }
  };
```

- **`getOrgId()`** — Tries two strategies to find your organization UUID. First it checks the `lastActiveOrg` browser cookie. If that's missing, it scans the browser's resource timing entries (the network request log) looking for any URL that contains `/api/organizations/{uuid}`. This two-pronged approach handles different browser states.
- **`getProjectId()`** — Extracts the project UUID directly from the current page URL using a regex match on `/project/{uuid}`. Returns `null` if not on a project page.
- **`timestamp()`** — Produces a filesystem-safe ISO timestamp like `2026-03-14T12-00-00` for use in filenames.
- **`sleep(ms)`** — A Promise-based delay, used to implement rate limiting between API calls.
- **`safeName(n)`** — Strips special characters from a name to make it safe for use as a directory or filename, then truncates to 60 characters.

## 4. ApiClient (lines 41–53)

A minimal HTTP client that wraps `fetch` with the organization's base URL and the user's session credentials.

```bash
sed -n '41,53p' export-claude-max.js
```

```output
  function ApiClient(orgId) {
    this.base = CONFIG.API_BASE + '/organizations/' + orgId;
  }
  ApiClient.prototype.get = function(path) {
    var url = this.base + path;
    return fetch(url, {
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' }
    }).then(function(r) {
      if (!r.ok) throw new Error('API ' + r.status + ': ' + path);
      return r.json();
    });
  };
```

The constructor builds the base URL: `/api/organizations/{orgId}`. The single method `.get(path)` appends a path to that base and calls `fetch`. Two things make this work in the browser:
- **`credentials: 'include'`** — sends the session cookies with every request, so Claude.ai's API sees you as a logged-in user
- The method throws an error on non-2xx status codes, and returns the parsed JSON body on success

All API calls in this codebase go through this one method.

## 5. paginatedFetch (lines 56–84)

Most Claude.ai API endpoints return results in pages. This function handles that transparently for callers.

```bash
sed -n '56,84p' export-claude-max.js
```

```output
  function paginatedFetch(api, basePath, pageSize) {
    var all = [];
    var offset = 0;
    function fetchPage() {
      var sep = basePath.indexOf('?') >= 0 ? '&' : '?';
      var path = basePath + sep + 'limit=' + pageSize + '&offset=' + offset;
      return api.get(path).then(function(resp) {
        // Handle both { data: [], pagination: {} } and plain array responses
        var items, hasMore;
        if (resp && Array.isArray(resp.data)) {
          items = resp.data;
          hasMore = resp.pagination && resp.pagination.has_more;
        } else if (Array.isArray(resp)) {
          items = resp;
          hasMore = items.length >= pageSize;
        } else {
          items = [];
          hasMore = false;
        }
        all = all.concat(items);
        if (hasMore) {
          offset += pageSize;
          return Utils.sleep(80).then(fetchPage);
        }
        return all;
      });
    }
    return fetchPage();
  }
```

The function accumulates results across pages using a recursive inner function `fetchPage()`. It appends `limit=` and `offset=` query parameters to the base path (handling whether a `?` is already present).

The response normalization handles two different API shapes Claude.ai uses:
1. **Object with `data` array** — uses `pagination.has_more` flag to decide whether to continue
2. **Plain array** — infers there are more pages if the returned count equals the page size (a common heuristic)

An 80ms sleep between pages avoids hammering the API too fast.

## 6. Extractors (lines 88–196)

Seven functions that each call specific API endpoints. The first four are simple wrappers:

```bash
sed -n '88,117p' export-claude-max.js
```

```output
  function extractProjectList(api) {
    return paginatedFetch(api, '/projects_v2?is_archived=false', CONFIG.PAGE_SIZE);
  }

  function extractProjectMeta(api, projectId) {
    return api.get('/projects/' + projectId).then(function(d) {
      return {
        id: d.uuid || projectId,
        name: d.name || '',
        description: d.description || '',
        instructions: d.prompt_template || d.custom_instructions || '',
        model: d.model || '',
        created_at: d.created_at || null,
        updated_at: d.updated_at || null,
        is_starred: !!d.is_starred
      };
    });
  }

  function extractProjectChatList(api, projectId) {
    return paginatedFetch(api, '/projects/' + projectId + '/conversations_v2', CONFIG.PAGE_SIZE);
  }

  function extractOrphanChatList(api) {
    return paginatedFetch(api, '/chat_conversations', CONFIG.PAGE_SIZE);
  }

  function extractChatFull(api, chatId) {
    return api.get('/chat_conversations/' + chatId);
  }
```

- **`extractProjectList`** — fetches all non-archived projects via the `/projects_v2` endpoint with paginated results
- **`extractProjectMeta`** — fetches one project's metadata and normalizes it, checking multiple possible field names (e.g. `prompt_template` or `custom_instructions`) since the API has evolved over time
- **`extractProjectChatList`** — lists all conversations in a specific project
- **`extractOrphanChatList`** — lists all conversations at the account level (includes both project and non-project chats)
- **`extractChatFull`** — fetches a single conversation with its complete message history

Next, the more complex file extractor:

```bash
sed -n '119,153p' export-claude-max.js
```

```output
  function extractProjectFiles(api, projectId) {
    // Fetch both text docs (with content) and blob files (metadata only)
    var docsPromise = api.get('/projects/' + projectId + '/docs').then(function(docs) {
      return (docs || []).map(function(d) {
        return {
          id: d.uuid, filename: d.file_name || 'unknown',
          content: d.content || '', content_type: d.content_type || '',
          created_at: d.created_at, file_size: d.file_size || 0,
          file_kind: 'doc'
        };
      });
    }).catch(function() { return []; });

    var blobsPromise = api.get('/projects/' + projectId + '/files').then(function(files) {
      return (files || []).map(function(f) {
        return {
          id: f.uuid || f.file_uuid, filename: f.file_name || 'unknown',
          content: '', content_type: '',
          created_at: f.created_at, file_size: f.size_bytes || 0,
          file_kind: f.file_kind || 'blob',
          path: f.path || ''
        };
      });
    }).catch(function() { return []; });

    return Promise.all([docsPromise, blobsPromise]).then(function(results) {
      var docs = results[0];
      var blobs = results[1];
      // Deduplicate: if a file appears in both, prefer the doc version (has content)
      var docIds = {};
      for (var i = 0; i < docs.length; i++) { docIds[docs[i].id] = true; }
      var uniqueBlobs = blobs.filter(function(b) { return !docIds[b.id]; });
      return docs.concat(uniqueBlobs);
    });
  }
```

**`extractProjectFiles`** fetches from two different endpoints in parallel using `Promise.all`:

1. **`/docs`** — returns text-based knowledge files with their full `content` field included. These are files you've uploaded as plain text that Claude can read.
2. **`/files`** — returns binary blobs (images, PDFs, etc.) with only metadata, no content.

After both promises resolve, the function deduplicates: if the same file ID appears in both lists, the doc version is preferred (it has content). This handles cases where text files appear in both endpoints. Both fetches are wrapped in `.catch(() => [])` so a failure on either endpoint doesn't abort the whole export.

### mapChatMessages and fetchFullChats (lines 155–196)

These two functions handle converting raw API data and sequentially fetching all chats.

```bash
sed -n '155,196p' export-claude-max.js
```

```output
  function mapChatMessages(f) {
    return {
      id: f.uuid,
      name: f.name || '',
      project_uuid: f.project_uuid || null,
      created_at: f.created_at,
      updated_at: f.updated_at,
      model: f.model || '',
      messages: (f.chat_messages || []).map(function(m) {
        return {
          role: m.sender,
          text: m.text || '',
          content: m.content || null,
          created_at: m.created_at,
          attachments: (m.attachments || []).map(function(a) {
            return { id: a.id, file_name: a.file_name, file_type: a.file_type, file_size: a.file_size };
          })
        };
      })
    };
  }

  function fetchFullChats(api, chatList, onProgress) {
    var results = [];
    var i = 0;
    function next() {
      if (i >= chatList.length) return Promise.resolve(results);
      var c = chatList[i];
      var label = c.name || 'Chat ' + (i + 1);
      if (onProgress) onProgress('Chat ' + (i + 1) + '/' + chatList.length + ': ' + label);
      return extractChatFull(api, c.uuid).then(function(f) {
        results.push(mapChatMessages(f));
      }).catch(function(e) {
        results.push({ id: c.uuid, name: label, project_uuid: c.project_uuid || null, error: e.message, messages: [] });
      }).then(function() {
        i++;
        if (i < chatList.length) return Utils.sleep(CONFIG.CHAT_DELAY_MS).then(next);
        return results;
      });
    }
    return next();
  }
```

**`mapChatMessages`** normalizes the raw API chat object. The API uses `sender` for the message role and `chat_messages` as the array key; this function renames them to standard `role` and `messages` and keeps both `text` (plain text) and `content` (structured content blocks) to handle different message formats.

**`fetchFullChats`** iterates a list of chat stubs and calls `extractChatFull` for each. Key design choices:
- **Sequential not parallel** — chats are fetched one at a time with a 750ms delay between them to avoid rate limiting
- **Error resilience** — if fetching a single chat fails, it pushes a stub object with the error message instead of aborting the entire export
- **Progress callbacks** — the `onProgress` function is called before each fetch so the UI can show which chat is being loaded

## 7. Data Assemblers (lines 200–317)

Three functions that orchestrate the extractors into complete export payloads.

```bash
sed -n '200,219p' export-claude-max.js
```

```output
  function assembleCurrentProject(api, projectId, onProgress) {
    var meta, chats, files;
    onProgress('Fetching project metadata...');
    return extractProjectMeta(api, projectId).then(function(m) {
      meta = m;
      onProgress('Listing project chats...');
      return extractProjectChatList(api, projectId);
    }).then(function(chatList) {
      onProgress('Found ' + chatList.length + ' chats in this project');
      return fetchFullChats(api, chatList, onProgress);
    }).then(function(c) {
      chats = c;
      onProgress('Fetching project files...');
      return extractProjectFiles(api, projectId);
    }).then(function(f) {
      files = f;
      var proj = Object.assign({}, meta, { chats: chats, files: files });
      return wrapExport([proj], []);
    });
  }
```

**`assembleCurrentProject`** is a linear Promise chain: metadata → chat list → full chats → files → wrap. It uses closure variables (`meta`, `chats`, `files`) to accumulate results across `.then()` callbacks, then merges them into one project object via `Object.assign` and passes it to `wrapExport` as a single-element array.

```bash
sed -n '221,285p' export-claude-max.js
```

```output
  function assembleAll(api, onProgress) {
    var projectsMeta = [];
    var allProjectChats = {};
    var orphanChats = [];
    var rawProjects;

    onProgress('Listing all projects...');
    return extractProjectList(api).then(function(rp) {
      rawProjects = rp;
      onProgress('Found ' + rawProjects.length + ' projects');

      // Fetch each project's chats + metadata + files sequentially
      var i = 0;
      function nextProject() {
        if (i >= rawProjects.length) return Promise.resolve();
        var rp = rawProjects[i];
        var pid = rp.uuid;
        onProgress('Project ' + (i + 1) + '/' + rawProjects.length + ': ' + (rp.name || 'Untitled'));

        return extractProjectMeta(api, pid).then(function(meta) {
          return extractProjectChatList(api, pid).then(function(chatList) {
            onProgress('  ' + chatList.length + ' chats');
            return fetchFullChats(api, chatList, onProgress).then(function(chats) {
              return extractProjectFiles(api, pid).then(function(files) {
                projectsMeta.push(Object.assign({}, meta, { chats: chats, files: files }));
              });
            });
          });
        }).catch(function(e) {
          projectsMeta.push({
            id: pid, name: rp.name || 'Untitled',
            description: '', instructions: '', model: '',
            created_at: rp.created_at, updated_at: rp.updated_at,
            error: e.message, chats: [], files: []
          });
        }).then(function() {
          i++;
          if (i < rawProjects.length) return Utils.sleep(CONFIG.PROJECT_DELAY_MS).then(nextProject);
        });
      }

      return nextProject();
    }).then(function() {
      // Fetch non-project chats
      onProgress('Fetching non-project chats...');
      return extractOrphanChatList(api);
    }).then(function(orphanList) {
      // Filter out chats that belong to any project
      var projectChatIds = {};
      for (var p = 0; p < projectsMeta.length; p++) {
        var pchats = projectsMeta[p].chats || [];
        for (var c = 0; c < pchats.length; c++) {
          projectChatIds[pchats[c].id] = true;
        }
      }
      var trueOrphans = orphanList.filter(function(c) { return !projectChatIds[c.uuid]; });
      onProgress('Found ' + trueOrphans.length + ' non-project chats');

      if (trueOrphans.length === 0) return [];
      return fetchFullChats(api, trueOrphans, onProgress);
    }).then(function(oc) {
      orphanChats = oc || [];
      return wrapExport(projectsMeta, orphanChats);
    });
  }
```

**`assembleAll`** is the more complex assembly path. It uses the same recursive inner-function pattern as `paginatedFetch` — a `nextProject()` closure that processes projects one at a time with a 1-second delay between them. Error handling is project-level: if one project fails entirely, it pushes a stub with an error field rather than aborting.

The orphan chat logic is important: `extractOrphanChatList` returns **all** conversations at the account level, which includes project chats. To find the true standalone (orphan) chats, it builds a set of all IDs already seen in project chat lists, then filters the full list to exclude them. This deduplication is necessary because the `/chat_conversations` endpoint doesn't have a filter for 'non-project only'.

```bash
sed -n '287,317p' export-claude-max.js
```

```output
  function wrapExport(projects, orphanChats) {
    var totalChats = orphanChats.length;
    var totalMessages = orphanChats.reduce(function(s, c) { return s + (c.messages || []).length; }, 0);
    var totalFiles = 0;
    var erroredChats = orphanChats.filter(function(c) { return c.error; }).length;

    for (var i = 0; i < projects.length; i++) {
      var p = projects[i];
      totalChats += (p.chats || []).length;
      totalMessages += (p.chats || []).reduce(function(s, c) { return s + (c.messages || []).length; }, 0);
      totalFiles += (p.files || []).filter(function(f) { return !f.error; }).length;
      erroredChats += (p.chats || []).filter(function(c) { return c.error; }).length;
    }

    return {
      _format: 'claude-project-export',
      _version: CONFIG.VERSION,
      exported_at: new Date().toISOString(),
      source_url: window.location.href,
      projects: projects,
      orphan_chats: orphanChats,
      stats: {
        total_projects: projects.length,
        total_chats: totalChats,
        total_messages: totalMessages,
        total_files: totalFiles,
        errored_chats: erroredChats,
        orphan_chats: orphanChats.length
      }
    };
  }
```

**`wrapExport`** adds a standard envelope around the assembled data. The `_format` and `_version` fields make the JSON self-describing, and the `stats` block gives a quick summary (also displayed in the UI completion dialog). This is the final shape of the data object — everything from here on just serializes or formats this structure.

## 8. Markdown Formatters (lines 321–424)

The `Markdown` object contains three functions that convert the export data into human-readable Markdown files.

```bash
sed -n '322,350p' export-claude-max.js
```

```output
    formatChat: function(chat) {
      var lines = ['---', 'title: ' + JSON.stringify(chat.name || 'Untitled'), 'id: ' + chat.id];
      if (chat.project_uuid) lines.push('project: ' + chat.project_uuid);
      if (chat.model) lines.push('model: ' + chat.model);
      if (chat.created_at) lines.push('created: ' + chat.created_at);
      if (chat.updated_at) lines.push('updated: ' + chat.updated_at);
      lines.push('messages: ' + (chat.messages || []).length, '---', '', '# ' + (chat.name || 'Untitled Chat'), '');
      if (chat.error) return lines.concat(['> **Export Error:** ' + chat.error, '']).join('\n');
      var msgs = chat.messages || [];
      for (var i = 0; i < msgs.length; i++) {
        var msg = msgs[i];
        var label = msg.role === 'human' ? 'Human' : msg.role === 'assistant' ? 'Assistant' : msg.role;
        lines.push('## ' + label, '');
        if (msg.text) { lines.push(msg.text); }
        else if (Array.isArray(msg.content)) {
          for (var j = 0; j < msg.content.length; j++) {
            var b = msg.content[j];
            if (b && b.type === 'text') lines.push(b.text || '');
          }
        }
        lines.push('');
        if (msg.attachments && msg.attachments.length) {
          for (var k = 0; k < msg.attachments.length; k++) lines.push('\u{1F4CE} `' + msg.attachments[k].file_name + '`');
          lines.push('');
        }
        lines.push('---', '');
      }
      return lines.join('\n');
    },
```

**`formatChat`** produces a single Markdown file per conversation. It starts with a YAML frontmatter block (between `---` delimiters) containing machine-readable metadata like IDs and timestamps. Then each message becomes an `## Human` or `## Assistant` section.

Message text is extracted with a fallback: first try `msg.text` (plain string), then fall back to iterating `msg.content` blocks looking for `type: 'text'` entries. This handles both old and new Claude API message formats. Attachments are listed below the message text as emoji-prefixed filenames.

```bash
sed -n '352,393p' export-claude-max.js
```

```output
    formatProjectReadme: function(project, exportData) {
      var lines = ['# ' + (project.name || 'Untitled Project'), ''];
      if (project.description) lines.push(project.description, '');
      lines.push('| Field | Value |', '|-------|-------|');
      lines.push('| **ID** | `' + project.id + '` |');
      if (project.model) lines.push('| **Model** | `' + project.model + '` |');
      if (project.created_at) lines.push('| **Created** | ' + project.created_at + ' |');
      if (project.updated_at) lines.push('| **Updated** | ' + project.updated_at + ' |');
      if (exportData) lines.push('| **Exported** | ' + exportData.exported_at + ' |');
      lines.push('');
      if (project.instructions) lines.push('## Custom Instructions', '', '```', project.instructions, '```', '');
      var chats = project.chats || [];
      if (chats.length) {
        lines.push('## Chats (' + chats.length + ')', '');
        for (var i = 0; i < chats.length; i++) {
          var c = chats[i];
          var fname = String(i + 1).padStart(3, '0') + '-' + Utils.safeName(c.name || 'Untitled') + '.md';
          var n = (c.messages || []).length;
          var d = (c.created_at || '').slice(0, 10);
          lines.push('- [' + (c.name || 'Untitled') + '](chats/' + fname + ') \u2014 ' + n + ' messages' + (d ? ' \u00B7 ' + d : ''));
        }
        lines.push('');
      }
      var files = (project.files || []).filter(function(f) { return !f.error; });
      var docFiles = files.filter(function(f) { return f.file_kind !== 'blob'; });
      var blobFiles = files.filter(function(f) { return f.file_kind === 'blob'; });
      if (docFiles.length) {
        lines.push('## Knowledge Files (' + docFiles.length + ')', '');
        for (var j = 0; j < docFiles.length; j++) lines.push('- [' + docFiles[j].filename + '](files/' + docFiles[j].filename + ')');
        lines.push('');
      }
      if (blobFiles.length) {
        lines.push('## Uploaded Files \u2014 metadata only (' + blobFiles.length + ')', '');
        for (var bj = 0; bj < blobFiles.length; bj++) {
          var bf = blobFiles[bj];
          var sz = bf.file_size ? ' (' + bf.file_size + ' bytes)' : '';
          lines.push('- `' + bf.filename + '`' + sz);
        }
        lines.push('');
      }
      return lines.join('\n');
    },
```

**`formatProjectReadme`** generates the `README.md` that sits at the root of each project folder inside the ZIP. It contains a metadata table, the project's custom system instructions (in a code block), a linked list of all chat files using zero-padded numbered filenames like `001-Chat-Name.md`, and two separate file sections: knowledge files (which have content and are linked) versus uploaded blobs (which only have metadata and are listed as unlinked items with their file sizes).

```bash
sed -n '395,424p' export-claude-max.js
```

```output
    formatRootReadme: function(data) {
      var s = data.stats;
      var lines = ['# Claude Export', '', 'Exported: ' + data.exported_at, '', '## Stats', '',
        '- **Projects:** ' + s.total_projects,
        '- **Total chats:** ' + s.total_chats,
        '- **Total messages:** ' + s.total_messages,
        '- **Knowledge files:** ' + s.total_files];
      if (s.orphan_chats) lines.push('- **Non-project chats:** ' + s.orphan_chats);
      if (s.errored_chats) lines.push('- **Errored chats:** ' + s.errored_chats);
      lines.push('');
      if (data.projects.length) {
        lines.push('## Projects', '');
        for (var i = 0; i < data.projects.length; i++) {
          var p = data.projects[i];
          lines.push('- [' + (p.name || 'Untitled') + '](' + Utils.safeName(p.name) + '/) \u2014 ' + (p.chats || []).length + ' chats');
        }
        lines.push('');
      }
      if (data.orphan_chats.length) {
        lines.push('## Non-Project Chats (' + data.orphan_chats.length + ')', '');
        for (var j = 0; j < data.orphan_chats.length; j++) {
          var c = data.orphan_chats[j];
          var fname = String(j + 1).padStart(3, '0') + '-' + Utils.safeName(c.name || 'Untitled') + '.md';
          lines.push('- [' + (c.name || 'Untitled') + '](_no-project/chats/' + fname + ')');
        }
        lines.push('');
      }
      return lines.join('\n');
    }
  };
```

**`formatRootReadme`** is only added to the ZIP when there are multiple projects or orphan chats. It's the top-level index: export stats summary, then links to each project folder, then links to standalone chats stored in the `_no-project/` folder. The `_no-project` prefix ensures this directory sorts visually distinct from project folders.

## 9. ArtifactParser (lines 428–457)

Extracts code artifacts that Claude embedded in its responses.

```bash
sed -n '428,457p' export-claude-max.js
```

```output
  var ArtifactParser = {
    extract: function(msg) {
      var results = [];
      var text = msg.text || '';
      var re = /<antArtifact([^>]*)>([\s\S]*?)<\/antArtifact>/g;
      var attrRe = /(\w+)="([^"]*)"/g;
      var m;
      while ((m = re.exec(text)) !== null) {
        var attrs = {}, am;
        attrRe.lastIndex = 0;
        while ((am = attrRe.exec(m[1])) !== null) attrs[am[1]] = am[2];
        results.push({ filename: attrs.title || attrs.identifier || 'artifact', content: m[2].trim(), language: attrs.language || attrs.type || '' });
      }
      if (Array.isArray(msg.content)) {
        for (var i = 0; i < msg.content.length; i++) {
          var b = msg.content[i];
          if (b && b.type === 'tool_use' && (b.name || '').toLowerCase().indexOf('artifact') >= 0) {
            var inp = b.input || {};
            results.push({ filename: inp.title || inp.id || 'artifact', content: inp.content || '', language: inp.language || inp.type || '' });
          }
        }
      }
      return results;
    },
    guessExt: function(lang, name) {
      if (name.indexOf('.') >= 0) return '';
      var map = { python:'.py',javascript:'.js',typescript:'.ts',tsx:'.tsx',jsx:'.jsx',html:'.html',css:'.css',json:'.json',yaml:'.yaml',markdown:'.md',bash:'.sh',shell:'.sh',sql:'.sql',rust:'.rs',go:'.go',java:'.java',react:'.jsx',vue:'.vue',ruby:'.rb',php:'.php',swift:'.swift',cpp:'.cpp',c:'.c',toml:'.toml',xml:'.xml' };
      return map[(lang || '').toLowerCase()] || '.txt';
    }
  };
```

Claude embeds generated code in two ways depending on the API version, and `ArtifactParser` handles both:

1. **XML tags in `msg.text`** — older format: `<antArtifact language="python" title="script.py">...code...</antArtifact>`. A global regex finds all such blocks, then a second regex extracts the attributes (title, language, identifier) from the opening tag.

2. **`tool_use` blocks in `msg.content`** — newer format: structured content blocks where `type === 'tool_use'` and the block name contains 'artifact'. The artifact content is in `block.input.content`.

**`guessExt`** maps language names to file extensions so artifact files get the right extension when saved to the ZIP. If the filename already contains a dot it's left alone; otherwise it appends from the map (or `.txt` as fallback).

## 10. buildZip (lines 461–507)

Assembles all the data into a JSZip archive with a logical directory structure.

```bash
sed -n '461,507p' export-claude-max.js
```

```output
  function buildZip(data) {
    var zip = new JSZip();
    if (data.projects.length > 1 || data.orphan_chats.length) {
      zip.file('README.md', Markdown.formatRootReadme(data));
    }
    var usedDirs = {};
    for (var pi = 0; pi < data.projects.length; pi++) {
      var proj = data.projects[pi];
      var dir = Utils.safeName(proj.name || 'Untitled');
      if (usedDirs[dir]) dir += '-' + (proj.id || '').slice(0, 8);
      usedDirs[dir] = true;
      var folder = zip.folder(dir);
      folder.file('README.md', Markdown.formatProjectReadme(proj, data));
      var chats = proj.chats || [];
      for (var ci = 0; ci < chats.length; ci++) {
        var chat = chats[ci];
        var cfn = String(ci + 1).padStart(3, '0') + '-' + Utils.safeName(chat.name || 'Untitled') + '.md';
        folder.file('chats/' + cfn, Markdown.formatChat(chat));
        var msgs = chat.messages || [];
        for (var mi = 0; mi < msgs.length; mi++) {
          if (msgs[mi].role !== 'assistant') continue;
          var arts = ArtifactParser.extract(msgs[mi]);
          for (var ai = 0; ai < arts.length; ai++) {
            var sn = Utils.safeName(arts[ai].filename);
            var ext = ArtifactParser.guessExt(arts[ai].language, arts[ai].filename);
            if (ext && sn.slice(-ext.length) !== ext) sn += ext;
            folder.file('files/artifacts/' + sn, arts[ai].content);
          }
        }
      }
      var files = proj.files || [];
      for (var fi = 0; fi < files.length; fi++) {
        if (files[fi].error) continue;
        if (!files[fi].content) continue;
        folder.file('files/' + (files[fi].filename || 'unknown'), files[fi].content || '');
      }
    }
    if (data.orphan_chats.length) {
      var of = zip.folder('_no-project');
      for (var oi = 0; oi < data.orphan_chats.length; oi++) {
        var oc = data.orphan_chats[oi];
        var ofn = String(oi + 1).padStart(3, '0') + '-' + Utils.safeName(oc.name || 'Untitled') + '.md';
        of.file('chats/' + ofn, Markdown.formatChat(oc));
      }
    }
    return zip.generateAsync({ type: 'blob', compression: 'DEFLATE', compressionOptions: { level: 6 } });
  }
```

The resulting ZIP has this structure:

```
claude-export-all-TIMESTAMP.zip
├── README.md                     (root index — only if multiple projects or orphans)
├── Project-Name/
│   ├── README.md
│   ├── chats/
│   │   ├── 001-Chat-Name.md
│   │   └── 002-Another-Chat.md
│   └── files/
│       ├── artifacts/            (code Claude generated, extracted from messages)
│       │   └── script.py
│       └── knowledge-file.txt    (files you uploaded to the project)
└── _no-project/
    └── chats/
        └── 001-Standalone-Chat.md
```

Key details: directory name conflicts are resolved by appending the first 8 characters of the project UUID. Only assistant messages are scanned for artifacts (Claude is the one generating code, not the human). Knowledge files are only written to the ZIP if they have content (`files[fi].content` check) — blobs without content are listed in the README but not extracted. `zip.generateAsync` returns a Promise that resolves to a Blob.

## 11. Exporters (lines 511–546)

Functions that trigger browser file downloads.

```bash
sed -n '511,546p' export-claude-max.js
```

```output
  function downloadBlob(blob, filename) {
    var url = URL.createObjectURL(blob);
    var a = document.createElement('a');
    a.href = url; a.download = filename;
    document.body.appendChild(a); a.click(); a.remove();
    setTimeout(function() { URL.revokeObjectURL(url); }, 5000);
  }

  function exportFilename(data, ext) {
    var prefix = 'claude-export';
    var name = data.projects.length === 1 ? prefix + '-' + Utils.safeName(data.projects[0].name) : prefix + '-all';
    return name + '-' + Utils.timestamp() + ext;
  }

  function downloadJson(data) {
    var blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
    downloadBlob(blob, exportFilename(data, '.json'));
  }

  function downloadZip(data) {
    return buildZip(data).then(function(blob) {
      downloadBlob(blob, exportFilename(data, '.zip'));
    });
  }

  /* ═══════════════════ SCRIPT LOADER ═══════════════════ */

  function loadScript(url) {
    return new Promise(function(resolve, reject) {
      if (typeof JSZip !== 'undefined') return resolve();
      var s = document.createElement('script');
      s.src = url; s.onload = resolve;
      s.onerror = function() { reject(new Error('Failed to load JSZip from CDN')); };
      document.head.appendChild(s);
    });
  }
```

**`downloadBlob`** uses the standard browser download trick: create a temporary object URL from the Blob, create an `<a>` element with `download` attribute set to the filename, append it to the DOM, programmatically click it, then immediately remove the element. The object URL is revoked after 5 seconds to free memory.

**`exportFilename`** generates the download filename: `claude-export-{project-name}-{timestamp}.json` for single-project exports or `claude-export-all-{timestamp}.zip` for multi-project.

**`loadScript`** lazily loads JSZip from CDN by injecting a `<script>` tag into `document.head`. It first checks whether JSZip is already defined (e.g. if the bookmarklet was clicked twice) to avoid loading it twice.

## 12. Dialog UI (lines 550–592)

The initial modal that appears when the bookmarklet is clicked, letting the user choose scope and format.

```bash
sed -n '550,592p' export-claude-max.js
```

```output
  function showDialog(currentProjectId) {
    return new Promise(function(resolve) {
      var existing = document.getElementById('__cpe_dialog');
      if (existing) existing.remove();
      var el = document.createElement('div');
      el.id = '__cpe_dialog';
      var onProj = !!currentProjectId;
      function opt(group, value, label, desc, checked, disabled) {
        return '<label style="display:flex;align-items:center;gap:10px;padding:10px 14px;background:#0f172a;border-radius:10px;border:1px solid #1e293b;cursor:' + (disabled ? 'not-allowed' : 'pointer') + ';margin-bottom:6px;opacity:' + (disabled ? '.5' : '1') + '">'
          + '<input type="radio" name="' + group + '" value="' + value + '"' + (checked ? ' checked' : '') + (disabled ? ' disabled' : '') + ' style="accent-color:#818cf8">'
          + '<div><div style="font-size:13px;color:#e2e8f0">' + label + '</div>'
          + '<div style="font-size:11px;color:#64748b">' + desc + '</div></div></label>';
      }
      el.innerHTML = '<div style="position:fixed;inset:0;background:rgba(0,0,0,.6);z-index:99999;display:flex;align-items:center;justify-content:center;font-family:-apple-system,BlinkMacSystemFont,sans-serif">'
        + '<div style="background:#1a1a2e;color:#e0e0e0;border-radius:16px;padding:32px 36px;min-width:400px;max-width:480px;box-shadow:0 24px 48px rgba(0,0,0,.5)">'
        + '<div style="font-size:15px;font-weight:700;color:#818cf8;margin-bottom:4px">\u{1F4E6} Claude Project Exporter</div>'
        + '<div style="font-size:12px;color:#64748b;margin-bottom:20px">v' + CONFIG.VERSION + '</div>'
        + '<div style="margin-bottom:18px">'
        + '<div style="font-size:12px;font-weight:600;color:#94a3b8;text-transform:uppercase;letter-spacing:.5px;margin-bottom:8px">Scope</div>'
        + opt('__cpe_scope', 'current', 'Current project only', onProj ? 'Export the project you\u2019re viewing' : 'Navigate to a project page first', onProj, !onProj)
        + opt('__cpe_scope', 'all', 'All projects + non-project chats', 'Every project with its files, plus standalone chats', !onProj, false)
        + '</div>'
        + '<div style="margin-bottom:22px">'
        + '<div style="font-size:12px;font-weight:600;color:#94a3b8;text-transform:uppercase;letter-spacing:.5px;margin-bottom:8px">Format</div>'
        + opt('__cpe_fmt', 'json', 'JSON', 'Raw structured data \u2014 for scripting & reimport', true, false)
        + opt('__cpe_fmt', 'zip', 'ZIP of project files', 'Markdown chats, knowledge files, extracted artifacts', false, false)
        + '</div>'
        + '<div style="display:flex;gap:10px;justify-content:flex-end">'
        + '<button id="__cpe_cancel" style="padding:8px 20px;background:#1e293b;color:#94a3b8;border:1px solid #334155;border-radius:8px;cursor:pointer;font-size:13px">Cancel</button>'
        + '<button id="__cpe_go" style="padding:8px 24px;background:linear-gradient(135deg,#6366f1,#818cf8);color:#fff;border:none;border-radius:8px;cursor:pointer;font-size:13px;font-weight:600">Export</button>'
        + '</div></div></div>';

      document.body.appendChild(el);
      function close(result) { el.remove(); resolve(result); }
      el.querySelector('#__cpe_cancel').onclick = function() { close(null); };
      el.firstChild.addEventListener('click', function(e) { if (e.target === el.firstChild) close(null); });
      el.querySelector('#__cpe_go').onclick = function() {
        var scope = (el.querySelector('input[name="__cpe_scope"]:checked') || {}).value;
        var format = (el.querySelector('input[name="__cpe_fmt"]:checked') || {}).value;
        close({ scope: scope, format: format });
      };
    });
  }
```

`showDialog` returns a Promise that resolves with the user's choices (`{scope, format}`) or `null` if cancelled. The entire UI is injected as an HTML string into a new `<div>` appended to `document.body`. All styles are inline to avoid conflicts with Claude.ai's stylesheet.

The dialog is context-aware: if `currentProjectId` is null (not on a project page), the 'Current project only' radio button is disabled and greyed out. Clicking the dark backdrop closes the dialog (via the outer `firstChild` click listener with a `target` check). The dialog removes itself from the DOM when closed.

## 13. ProgressUI (lines 596–634)

A second modal that replaces the dialog during the export, showing live status.

```bash
sed -n '596,634p' export-claude-max.js
```

```output
  function ProgressUI() {
    this.el = document.createElement('div');
    this.el.id = '__cpe_progress';
    this.el.innerHTML = '<div style="position:fixed;inset:0;background:rgba(0,0,0,.55);z-index:99999;display:flex;align-items:center;justify-content:center;font-family:-apple-system,BlinkMacSystemFont,sans-serif">'
      + '<div style="background:#1a1a2e;color:#e0e0e0;border-radius:16px;padding:32px 40px;min-width:400px;max-width:520px;box-shadow:0 24px 48px rgba(0,0,0,.4);text-align:center">'
      + '<div style="font-size:14px;font-weight:600;color:#818cf8;margin-bottom:6px">EXPORTING</div>'
      + '<div id="__cpe_status" style="font-size:14px;margin:16px 0;min-height:24px;color:#d1d5db">Initializing...</div>'
      + '<div style="background:#2d2d44;border-radius:8px;height:6px;overflow:hidden;margin:16px 0">'
      + '<div id="__cpe_bar" style="background:linear-gradient(90deg,#818cf8,#a78bfa);height:100%;width:5%;transition:width .3s;border-radius:8px"></div></div>'
      + '<div id="__cpe_stats" style="font-size:12px;color:#9ca3af;margin-top:8px"></div>'
      + '<button id="__cpe_close" style="display:none;margin-top:16px;padding:8px 24px;background:#818cf8;color:#fff;border:none;border-radius:8px;cursor:pointer;font-size:13px">Close</button>'
      + '</div></div>';
  }
  ProgressUI.prototype.show = function() {
    var old = document.getElementById('__cpe_progress');
    if (old) old.remove();
    document.body.appendChild(this.el);
    var self = this;
    document.getElementById('__cpe_close').onclick = function() { self.hide(); };
  };
  ProgressUI.prototype.update = function(msg, pct) {
    var s = document.getElementById('__cpe_status');
    var b = document.getElementById('__cpe_bar');
    if (s) s.textContent = msg;
    if (b && pct != null) b.style.width = Math.min(100, pct) + '%';
  };
  ProgressUI.prototype.done = function(msg, stats) {
    this.update(msg, 100);
    var t = document.getElementById('__cpe_stats');
    if (t && stats) t.textContent = stats.total_projects + ' projects \u00B7 ' + stats.total_chats + ' chats \u00B7 ' + stats.total_messages + ' msgs \u00B7 ' + stats.total_files + ' files';
    document.getElementById('__cpe_close').style.display = 'inline-block';
  };
  ProgressUI.prototype.error = function(msg) {
    this.update('Error: ' + msg, 0);
    var b = document.getElementById('__cpe_bar');
    if (b) b.style.background = '#ef4444';
    document.getElementById('__cpe_close').style.display = 'inline-block';
  };
  ProgressUI.prototype.hide = function() { this.el.remove(); };
```

`ProgressUI` is a classic prototype-based class. The constructor builds the DOM element once; `show()` appends it. Four prototype methods update state:

- **`update(msg, pct)`** — sets status text and moves the progress bar (capped at 100%)
- **`done(msg, stats)`** — fills the bar to 100%, shows the stats line, reveals the Close button
- **`error(msg)`** — resets bar to 0% and turns it red, reveals Close button
- **`hide()`** — removes the element from the DOM

The Close button is hidden during export and revealed only by `done` or `error`, preventing premature dismissal.

## 14. The Orchestrator (lines 638–680)

The main entry point — the code that runs immediately when the IIFE executes.

```bash
sed -n '638,680p' export-claude-max.js
```

```output
  var projectId = Utils.getProjectId();
  showDialog(projectId).then(function(choice) {
    if (!choice) return;
    var ui = new ProgressUI();
    ui.show();

    var orgId = Utils.getOrgId();
    if (!orgId) { ui.error('Could not detect org ID. Try refreshing.'); return; }

    var p = Promise.resolve();
    if (choice.format === 'zip') {
      ui.update('Loading ZIP library...', 5);
      p = loadScript(CONFIG.JSZIP_CDN);
    }

    p.then(function() {
      ui.update('Connecting...', 10);
      var api = new ApiClient(orgId);
      var step = 0;
      var onProgress = function(msg) { step++; ui.update(msg, Math.min(10 + step * 3, 90)); };

      if (choice.scope === 'current') {
        if (!projectId) throw new Error('Not on a project page. Use "All projects" or navigate to one.');
        return assembleCurrentProject(api, projectId, onProgress);
      } else {
        return assembleAll(api, onProgress);
      }
    }).then(function(data) {
      ui.update('Preparing download...', 95);
      if (choice.format === 'zip') {
        return downloadZip(data).then(function() { return data; });
      } else {
        downloadJson(data);
        return data;
      }
    }).then(function(data) {
      ui.done('Export complete!', data.stats);
    }).catch(function(e) {
      console.error('[Claude Export]', e);
      ui.error(e.message);
    });
  });
})();
```

This is the top-level control flow. It runs synchronously up to `showDialog`, then everything proceeds through Promise chains:

1. **`Utils.getProjectId()`** — called immediately to know if we're on a project page
2. **`showDialog(projectId)`** — shows the modal; the rest waits for user input
3. If cancelled (`choice` is null), return early — nothing happens
4. **`Utils.getOrgId()`** — detected after the dialog (not before) so the progress UI can show an error if it fails
5. **JSZip loading** — if ZIP was selected, `loadScript` runs before the API calls so the library is ready when `buildZip` eventually calls `new JSZip()`
6. **Progress percentage** — the `onProgress` callback increments `step` on each call and maps it to a percentage range of 10–90%, leaving room for the initial setup (0–10%) and final download (90–100%)
7. **Download** — `downloadZip` returns a Promise (ZIP generation is async); `downloadJson` is synchronous so it's not awaited
8. **Single `.catch`** at the end of the chain handles any error from any step and shows it in the progress modal

## Summary

Here's how all the pieces connect end-to-end:

```
Bookmarklet click
  → IIFE runs immediately
  → getProjectId() reads the URL
  → showDialog() renders modal, waits for user

User clicks Export
  → getOrgId() reads cookie / network log
  → [if ZIP] loadScript() fetches JSZip from CDN
  → ApiClient created with org base URL

  → [current scope] assembleCurrentProject()
       extractProjectMeta() → API: /projects/{id}
       extractProjectChatList() → paginatedFetch → API: /projects/{id}/conversations_v2
       fetchFullChats() → loop: extractChatFull() × N with 750ms delays
       extractProjectFiles() → parallel: /docs + /files → deduplicated

  → [all scope] assembleAll()
       extractProjectList() → paginatedFetch → API: /projects_v2
       loop per project (1000ms delay):
         extractProjectMeta + extractProjectChatList + fetchFullChats + extractProjectFiles
       extractOrphanChatList() → paginatedFetch → API: /chat_conversations
       filter to true orphans → fetchFullChats()

  → wrapExport() adds envelope + stats

  → [JSON] JSON.stringify → Blob → downloadBlob()
  → [ZIP]  buildZip():
       Markdown.formatRootReadme() → root README
       per project: Markdown.formatProjectReadme() + Markdown.formatChat() per chat
         + ArtifactParser.extract() per assistant message → files/artifacts/
         + knowledge file content → files/
       orphan chats: formatChat() → _no-project/chats/
       zip.generateAsync() → Blob → downloadBlob()

  → ProgressUI.done() shows stats and Close button
```
