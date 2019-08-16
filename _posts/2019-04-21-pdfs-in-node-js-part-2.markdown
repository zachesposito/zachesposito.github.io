---
layout: post
title:  "Creating PDFs with Node.js - Part 2"
date:   2019-04-21
subtitle: Generating PDFs using Puppeteer and HummusJS
---

In [part 1][part-1-post] I explored how to use [Puppeteer](https://github.com/GoogleChrome/puppeteer) to generate a PDF from HTML. Once we have a PDF, though, how can we manipulate it? For example, what if we need to automatically generate an index? We would somehow need to identify which keywords appear on which pages, turn that into a list, then add that list onto the end of the PDF. Is there a way to do all that in Node?

These questions led me to [HummusJS](https://github.com/galkahana/HummusJS), a Node module for parsing and manipulating PDFs. Unlike other Node.js PDF manipulation modules such as [pdf-merge](https://github.com/wubzz/pdf-merge) or [node-pdftk](https://github.com/jjwilly16/node-pdftk), HummusJS doesn't require installing [PDFtk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/), so all you need to get started is the HummusJS Node module itself.

Here's how I used Hummus to append an index to a PDF:

## Step 1: parse the PDF

**parser.js**:
{% highlight javascript %}
const hummus = require('hummus');
const extractText = require('./text-extraction/lib/text-extraction');

/**
 * Get all text on all pages from PDF buffer
 * @param {Buffer} PDFBuffer
 * @returns {Array.<{text: string, matrix: Number[], localBBox: Number[], globalBBox: Number[]}[]>} - An array with an element for each page. Each element is an array of text placements on the page.
 */
module.exports.parsePDFBuffer = (PDFBuffer) => {
    var PDFStream = new hummus.PDFRStreamForBuffer(PDFBuffer);
    var pdfReader = hummus.createReader(PDFStream);
    return extractText(pdfReader);
};
{% endhighlight %}

`extractText` is the result of `require()`ing [this file](https://github.com/galkahana/HummusJSSamples/blob/master/text-extraction/lib/text-extraction.js), part of a text extraction sample from the [HummusJS author](https://github.com/galkahana). The module exported by **parser.js** simply wraps a [PDFReader](https://github.com/galkahana/HummusJS/blob/2ac4ad4b8ab9a43bcef49c9827cbf6014bc7e761/hummus.d.ts#L362) around a buffer containing the PDF data, which is then used by `extractText()`, which returns an array of text information for each page.

## Step 2: use parsed text to build index as an HTML list

**index.js**
{% highlight javascript %}
const parser = require('./parser');

/**
 * Append an index to a PDF
 * @param {Buffer} PDFBuffer - A Buffer containing the PDF to create an index for and append to.
 * @param {Array<string>} indexKeywords - an array of keywords to list in the index
 */
const appendIndex = async (PDFBuffer, indexKeywords) => {
    var pages = parser.parsePDFBuffer(PDFBuffer);
    var indexData = locateIndexKeywords(pages, indexKeywords);
    var indexHTML = getIndexHTML(indexData);
};

/**
 * Append an index to a PDF
 * @param {Array.<{text: string, matrix: Number[], localBBox: Number[], globalBBox: Number[]}[]>} pages - an array of text information for each page, see parser.parsePDFBuffer()
 * @param {Array<string>} indexKeywords - an array of keywords to locate
 * @returns - a dictionary of keyword to the number of the first page found on
 */
const locateIndexKeywords = (pages, indexKeywords) => {
    var foundKeywords = [];

    for (var keyword of indexKeywords){
        var keywordFound = false;
        var pageIndex = 0;

        while (!keywordFound && pageIndex < pages.length) {
            var currentPage = pages[pageIndex];
            var textBlockIndex = 0;

            while (!keywordFound && textBlockIndex < currentPage.length) {
                if (currentPage[textBlockIndex].text.includes(keyword)){
                    keywordFound = true;
                    foundKeywords.push({keyword: keyword, pageNumber: pageIndex + 1});
                }                          
                textBlockIndex++;
            }             
            pageIndex++;
        }        
    }

    return foundKeywords;
};

/**
 * 
 * @param {Array<{keyword: string, pageNumber: number}>} indexData - A dictionary of keyword to page number, see locateIndexKeywords()
 * @returns {string} - The index as HTML
 */
const getIndexHTML = (indexData) => {
    var html = '<html><head></head><body>';
    html = '<h1>Index</h1>';
    html += '<ol>'

    for (var entry of indexData){
        html += `<li><a href="#${entry.keyword}">${entry.keyword} - ${entry.pageNumber}</a></li>`;
    }

    html += '</ol>';
    html += '</body></html>'
    return html;
};
{% endhighlight %}

Here, `appendIndex()` accepts the PDF to create an index for and a list of keywords to include, then passes the result of `parser.parsePDFBuffer()` to `locateIndexKeywords()`, which finds the first occurrence of each keyword in the PDF and returns a dictionary of keyword to page number. Then, `getIndexHTML()` transforms that dictionary into an HTML list.

## Step 3: make a new PDF from the index HTML and append it to the original PDF

**combiner.js**
{% highlight javascript %}
const hummus = require('hummus');
const memoryStreams = require('memory-streams');

/**
 * Concatenate two PDFs in Buffers
 * @param {Buffer} firstBuffer 
 * @param {Buffer} secondBuffer 
 * @returns {Buffer} - a Buffer containing the concactenated PDFs
 */
module.exports.combinePDFBuffers = (firstBuffer, secondBuffer) => {
    var outStream = new memoryStreams.WritableStream();

    try {
        var firstPDFStream = new hummus.PDFRStreamForBuffer(firstBuffer);
        var secondPDFStream = new hummus.PDFRStreamForBuffer(secondBuffer);

        var pdfWriter = hummus.createWriterToModify(firstPDFStream, new hummus.PDFStreamForResponse(outStream));
        pdfWriter.appendPDFPagesFromPDF(secondPDFStream);
        pdfWriter.end();
        var newBuffer = outStream.toBuffer();
        outStream.end();

        return newBuffer;
    }
    catch(e){
        outStream.end();
        throw new Error('Error during PDF combination: ' + e.message);
    }
};
{% endhighlight %}

`combinePDFBuffers()` takes two buffers containing PDF data and appends the second to the first.

**index.js**

{% highlight javascript %}
const renderer = require('./renderer');
const combiner = require('./combiner');
const parser = require('./parser');

/**
 * Append an index to a PDF
 * @param {Buffer} PDFBuffer - A Buffer containing the PDF to create an index for and append to.
 * @param {Array<string>} indexKeywords - an array of keywords to list in the index
 */
const appendIndex = async (PDFBuffer, indexKeywords) => {
    var pages = parser.parsePDFBuffer(PDFBuffer);
    var indexData = locateIndexKeywords(pages, indexKeywords);
    var indexHTML = getIndexHTML(indexData);

    /* New: */
    var indexPDFBuffer = await renderer.renderHTMLtoPDF(indexHTML);
    return combiner.combinePDFBuffers(PDFBuffer, indexPDFBuffer);
};

const locateIndexKeywords = (pages, indexKeywords) => {/* ... */};
const getIndexHTML = (indexData) => { /* ... */};
{% endhighlight %}

`renderer` is **renderer.js** from [part 1][part-1-post]. Just like in part 1, its `rendererHTMLtoPDF()` method is used to turn the index's HTML into a buffer of PDF data. Finally, `combiner.combinePDFBuffers()` produces a new buffer containing the data of the original PDF plus the freshly rendered index PDF.

[part-1-post]:{{ site.baseurl }}{% post_url 2019-04-14-pdfs-in-node-js-part-1 %}