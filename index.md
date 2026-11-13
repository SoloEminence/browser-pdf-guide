---
layout: default
title: How browser-only PDF tools work
description: What actually happens when you merge, compress or edit a PDF inside your browser, and how to check that your file was never uploaded.
---

# How browser-only PDF tools work

Most free PDF sites follow the same routine. You upload the file, their server does the work, you download the result. So a copy of your bank statement or ID card sits on someone else's machine for a while, and you have to trust their deletion policy.

It doesn't have to work that way. A modern browser can do the whole job on your own device. I build one of these tools, so this page covers how it's done, where it falls short, and how to check any site's claim in about 30 seconds.

## The pieces

When you pick a file or drag it onto a page, JavaScript gets a `File` object. Calling `file.arrayBuffer()` hands the page the raw bytes. That read happens in memory. There is no network request.

Next comes a PDF library that runs in the browser. [pdf-lib](https://pdf-lib.js.org/) handles writing and editing, and Mozilla's [PDF.js](https://mozilla.github.io/pdf.js/) handles reading and rendering. Heavier work like OCR or image re-encoding usually runs as WebAssembly, which is close to native speed.

Finally the page wraps the new bytes in a `Blob`, makes a temporary `blob:` URL and triggers the download. The file goes from memory to your Downloads folder.

```js
const bytes = await file.arrayBuffer();
const pdf = await PDFDocument.load(bytes);
// merge, rotate, delete pages...
const out = await pdf.save();
const url = URL.createObjectURL(new Blob([out], { type: 'application/pdf' }));
```

Nothing in that chain talks to a server.

## What common jobs look like locally

| Task | What the browser does |
| --- | --- |
| Merge | Copies pages from each file into one new document and saves it. Fast and lossless. |
| Split, delete, reorder | Rewrites the page tree. The page content isn't touched. |
| Compress | Decodes the embedded images, scales them down and re-encodes them. This is where nearly all the savings come from. |
| Edit text | Parses the page into positioned text runs, removes the run you changed from the content stream, and redraws it, using the document's own embedded font when it can. |
| OCR | Renders each page to an image and runs a recognition model in a Web Worker so the tab stays responsive. |
| Protect, unlock | Applies or removes encryption with the password you type. The password stays on your machine. |

## Where it falls short

Everything lives in RAM. A 300-page scan can choke a phone, though a desktop browser copes with a few hundred MB.

OCR models and fonts weigh several MB, so the first load is slower. A decent tool caches them after that.

PDF to Word is always an approximation, local or not. A PDF stores glyphs at coordinates, not paragraphs, so the converter has to guess the structure.

## Checking a site yourself

1. Press F12 and open the Network tab.
2. Clear it, then process a file.
3. Look for a request roughly the size of your PDF. If all you see is scripts, fonts and analytics, your file stayed put.

A blunter test: load the tool, turn off Wi-Fi, then run it. If it still works, it isn't uploading anything.

## Tools built this way

[TryDocsy](https://trydocsy.com/) is the set I work on. Every tool runs locally and keeps working offline once the page has loaded.

- [Merge PDF](https://trydocsy.com/pdf/merge) combines files and lets you reorder pages.
- [Edit PDF](https://trydocsy.com/pdf/edit) changes the existing text in place instead of pasting a box over it.
- [Compress PDF](https://trydocsy.com/pdf/compress) shrinks scans and image-heavy files.
- [OCR PDF](https://trydocsy.com/pdf/ocr) makes scanned pages searchable.
- The [full PDF tool list](https://trydocsy.com/pdf).

Use whatever you like, but run the network tab test on it first.

Related: [compressing a PDF under 100 KB for form uploads](https://trydocsy.com/compress-pdf-100kb), [redacting a PDF so the text is really gone](https://trydocsy.com/pdf/redact).

## More notes

- [Getting a PDF under 100 KB or 200 KB for an online form](https://gist.github.com/SoloEminence/9d08340ea2451ff20392dd9bca18b80e)
- [Photo and signature sizes for online application forms](https://write.as/dq6t8j5cqaryb)
- [Aadhaar, PAN and ID documents for online forms: a checklist](https://docs.google.com/document/d/e/2PACX-1vSXAL6h-BP-huiCZqPst-EFpcqa3IIhIHOAG28rocrZ7UdheVaHdi3eC5YYo5DGE5cixpjNHCFBvViM/pub)
