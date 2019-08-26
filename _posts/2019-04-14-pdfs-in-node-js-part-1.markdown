---
layout: post
title:  "Creating PDFs with Node.js - Part 1"
date:   2019-04-14
subtitle: Generating PDFs using Puppeteer and HummusJS
categories: [technical, walkthrough]
tags: [node.js]
---
The ability to generate PDFs is a common business need. There are many PDF libraries for a variety of programming languages and use cases available to help with this task. Some are proprietary and some are open-source.

In one of my recent projects I needed to create a PDF generation service using Node.js. My requirements were:
* PDF generation should be quick (less than a minute)
* Use only open-source libraries
* Layout the PDF contents using HTML and CSS (as opposed to a library-specific API)

These requirements led me to [Puppeteer](https://github.com/GoogleChrome/puppeteer). Puppeteer is an API for manipulating [Chromium](https://www.chromium.org/Home), which is an open-source version of the Chrome browser. The feature most relevant to my PDF generator project was the ability to create a PDF from a web page. 

Here is the PDF creation example from the repo's readme:

{% highlight javascript %}
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://news.ycombinator.com', {waitUntil: 'networkidle2'});
  await page.pdf({path: 'hn.pdf', format: 'A4'});

  await browser.close();
})();
{% endhighlight %}

The steps are essentially create a new browser instance, tell that instance to load a URL, then save the rendered page as a PDF. Easy!

Here's how I used Puppeteer in my project:

**renderer.js**:
{% highlight javascript %}
const puppeteer = require('puppeteer');
/**
 * Turn HTML into PDF as Buffer
 * @param {string} markup - HTML string to render to PDF
 * @returns {Promise<Buffer>} - PDF Buffer
 */
module.exports.renderHTMLtoPDF = async (markup) => {    
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    try {
        console.log('Loading HTML into page.');
        await page.setContent(markup);

        console.log('Rendering PDF.');
        var PDF = await page.pdf();
        await browser.close();
        return PDF;
    }
    catch(e){
        await browser.close();
        throw new Error('Error during PDF rendering: ' + e.message);
    }
};
{% endhighlight %}

Instead of getting the HTML into Chromium via a URL, I wanted to provide the HTML as a string. With Puppeteer you can set a page's HTML using [`page.setContent()`](https://github.com/GoogleChrome/puppeteer/blob/v1.14.0/docs/api.md#pagesetcontenthtml-options). After the page's HTML is set, it gets saved to a PDF with [`page.pdf()`](https://github.com/GoogleChrome/puppeteer/blob/v1.14.0/docs/api.md#pagepdfoptions).

**index.js**:
{% highlight javascript %}
const renderer = require('./renderer');
const fs = require('fs-extra');
const path = require('path');

(async () => {    
    var destinationFolder = 'output';
    try {
        var html = await fs.readFile('markup.html', 'utf8');
        var PDFBuffer = await renderer.generatePDFFromHTML(html);
        console.log('Got buffer of PDF data');

        if (!fs.existsSync(destinationFolder)) {
            fs.mkdirSync(destinationFolder);
        }
        
        await fs.writeFile(path.join(destinationFolder, 'moduletest.pdf'), PDFBuffer);
        console.log('Saved PDF');
    }
    catch(e){
        console.log(e.message);
        console.log(e.stack);
    }
})();
{% endhighlight %}

The index script reads HTML from a file (`markup.html`) and passes the HTML as a string to the renderer's `generatePDFFromHTML()` method, defined above. That method returns the PDF as a [`Buffer`](https://nodejs.org/api/buffer.html#buffer_buffer), which is then written to a file.

