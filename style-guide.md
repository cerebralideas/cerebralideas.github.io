---
layout: page
title: Style Guide
featured_image: /assets/images/covers/8.jpeg
---

The style guide provides you with a blueprint of Subtle default post and page styles. The style guide is also a great reference for suggested typographic treatment and styles for your content.

## Images

This theme supports **regular image with or without caption**, **wide images with or without caption** and **full-width images with or without the caption**. The examples provided below.

{% include image-caption.html imageurl="/assets/images/pages/regular.jpg" title="Concrete roads" caption="This regular image with caption" %}

{% include image-wide.html imageurl="/assets/images/pages/wide.jpeg" title="Wide image with caption" caption="This wide image with caption" %}

{% include image-full.html imageurl="/assets/images/pages/full.jpeg" title="Full-width image with caption" caption="This full-width image with caption" %}

One neat trick which you can use in Markdown to distinguish between different types of images is to add a #hash value to the end of the source URL, and then target images containing the hash with special styling. For example, to add a wider image, you need to add the #wide value to the end of the source URL.

## Align Image Left

<div class="alignleft">
	<img src="/assets/images/pages/left.jpg" alt="left">
</div>

One neat trick which you can use in Markdown to distinguish between different types of images is to add a #hash value to the end of the source URL, and then target images containing the hash with special styling. For example, to add a wider image, you need to add the #wide value to the end of the source URL.

Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Lorem ipsum dolor sit amet, consectetur adipiscing elit. Cras pulvinar tincidunt turpis ac fringilla. Mauris eget dolor neque. Donec ac ex quis lectus sagittis vestibulum.

Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat.

## Align Image Right

<div class="alignright">
	<img src="/assets/images/pages/right.jpg" alt="left">
</div>

Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Nulla libero. 

# This is an H1

<mark>Quisque facilisis erat a dui</mark>. Nam malesuada ornare dolor. Cras gravida, diam sit amet rhoncus ornare, erat elit consectetuer erat, id egestas pede nibh eget odio. Proin tincidunt, velit vel porta elementum, magna diam molestie sapien, non aliquet massa pede eu diam. Aliquam iaculis. Fusce et ipsum et nulla tristique facilisis.

## This is an H2

Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Nulla libero.

### This is an H3

Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Nulla libero.

#### This is an H4

Quisque facilisis erat a dui. Nam malesuada ornare dolor. Cras gravida, diam sit amet rhoncus ornare, erat elit consectetuer erat, id egestas pede nibh eget odio. Proin tincidunt, velit vel porta elementum, magna diam molestie sapien, non aliquet massa pede eu diam. Aliquam iaculis.

## Quoting

Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Nulla libero.

>“Creativity is allowing yourself to make mistakes. Design is knowing which ones to keep.” <cite>― Scott Adams ―</cite>

Lorem ipsum dolor sit amet, `consectetuer adipiscing` elit. Morbi commodo, ipsum sed pharetra gravida, orci magna rhoncus neque, id pulvinar odio lorem non turpis. Nullam sit amet enim. Suspendisse id velit vitae ligula volutpat condimentum. Aliquam erat volutpat. Sed quis velit. Nulla facilisi. Nulla libero.

--------

## Code Blocks

```
// Simple map
var map;
function initMap() {
  map = new google.maps.Map(document.getElementById('map'), {
    center: {lat: -34.397, lng: 150.644},
    zoom: 8
  });
}
```

Code blocks, highlighted with [Prism](http://prismjs.com/index.html).

<pre><code class="language-markup">&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
	&lt;meta charset="utf-8" /&gt;
	&lt;title&gt;Prism&lt;/title&gt;
	&lt;link rel="stylesheet" href="style.css" /&gt;
&lt;/head&gt;
&lt;body&gt;
	&lt;header&gt;
		&lt;div class="intro" data-src="templates/header-main.html" data-type="text/html"&gt;&lt;/div&gt;
		&lt;ul id="features"&gt;
			&lt;li&gt;Dead simple&lt;/li&gt;
			&lt;li&gt;Light as a feather&lt;/li&gt;
			&lt;li&gt;Extensible&lt;/li&gt;
		&lt;/ul&gt;
	&lt;/header&gt;
	&lt;section id="features-full" class="language-markup"&gt;
		&lt;h1&gt;Full list of features&lt;/h1&gt;
		&lt;ul&gt;
			&lt;li&gt;&lt;strong&gt;Only 2KB&lt;/strong&gt; minified &amp; gzipped (core). Each language definition adds roughly 300-500 bytes.&lt;/li&gt;
			&lt;li&gt;Very easy to extend without modifying the code, due to Prism’s &lt;a href="#plugins"&gt;plugin architecture&lt;/a&gt;. Multiple hooks are scattered throughout the source.&lt;/li&gt;
			&lt;li&gt;Very easy to &lt;a href="extending.html#language-definitions"&gt;define new languages&lt;/a&gt;. Only thing you need is a good understanding of regular expressions&lt;/li&gt;
		&lt;/ul&gt;
	&lt;/section&gt;
	&lt;footer data-src="templates/footer.html" data-type="text/html"&gt;&lt;/footer&gt;
&lt;/body&gt;
&lt;/html&gt;
</code></pre>

<pre><code class="language-javascript">var Token = _.Token = function(type, content, alias, matchedStr, greedy) {
	this.type = type;
	this.content = content;
	this.alias = alias;
	// Copy of the full string this token was created from
	this.length = (matchedStr || "").length|0;
	this.greedy = !!greedy;
};</code></pre>

<pre><code class="language-css">@import url(http://fonts.googleapis.com/css?family=Arvo);

/* Styles */

body {
	font: 100%/1.5 Questrial, sans-serif;
	tab-size: 4;
	hyphens: auto;
}

a {
	color: inherit;
}

section h1 {
	font-size: 250%;
}

section h1,
#features li strong,
header h2,
footer p {
	font: 100% Rockwell, Arvo, serif;
}</code></pre>

## Lists
### Unordered Lists

The HTML &lt;ul&gt; element represents an unordered list of items, typically rendered as a bulleted list.

Usage notes:
* The &lt;ul&gt; element is for grouping a collection of items that do not have a numerical ordering, and their order in the list is meaningless. Typically, unordered-list items are displayed with a bullet, which can be of several forms, like a dot, a circle or a squared.
* There is no limitation to the depth and alternation of nested lists defined with the &lt;ol&gt; and &lt;ul&gt; elements.
* The &lt;ol&gt; and &lt;ul&gt; elements both represent a list of items. They differ in that, with the &lt;ol&gt; element, the order is meaningful.

### Ordered Lists

The HTML &lt;ol&gt; element represents an ordered list of items, typically rendered as a numbered list.

Usage notes:
1. Typically, ordered-list items are displayed with a preceding numbering, which can be of any form, like numerals, letters or Romans numerals or even simple bullets.
2. There is no limitation to the depth and alternation of nested lists defined with the &lt;ol&gt; and &lt;ul&gt; elements.
3. The &lt;ol&gt; and &lt;ul&gt; both represent a list of items. They differ in the way that, with the &lt;ol&gt; element, the order is meaningful.

## Footnotes

The quick brown fox[^1] jumped over the lazy dog[^2].

[^1]: Foxes are red
[^2]: Dogs are usually not red

Footnotes are a great way to add additional contextual details when appropriate. Ghost will automatically add footnote content to the very end of your post.

## Tables

<table>
<caption>Table Caption</caption>
<thead>
<tr>
   <th>Content categories</th>
   <th>Flow content</th>
  </tr>
</thead>
 <tbody>
  <tr>
   <td>Permitted content</td>
   <td>
    An optional <code>&lt;caption&gt;</code> element;<br />
    zero or more <code>&lt;colgroup&gt;</code> elements;<br />
    an optional <code>&lt;thead&gt;</code> element;<br />
    an optional <code>&lt;tfoot&gt;</code> element;
   </td>
  </tr>
  <tr>
   <td>Tag omission</td>
   <td>None, both the start tag and the end tag are mandatory</td>
  </tr>
  <tr>
   <td>Permitted parent elements</td>
   <td>Any element that accepts flow content</td>
  </tr>
  <tr>
   <td>Normative document</td>
   <td><a href="http://www.whatwg.org/specs/web-apps/current-work/multipage/tabular-data.html#the-table-element" rel="external nofollow">HTML5, section 4.9.1</a> (<a href="http://www.w3.org/TR/REC-html40/struct/tables.html#edef-TABLE">HTML4.01, section 11.2.1</a>)</td>
  </tr>
 </tbody>
</table>

## Videos

<iframe src="https://player.vimeo.com/video/137643135?color=ffffff&title=0&byline=0&portrait=0" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="https://vimeo.com/137643135">This Mountain</a> from <a href="https://vimeo.com/evanmann">Evan Mann / OWP Denver</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

## Audio

<iframe width="100%" height="450" scrolling="no" frameborder="no" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/165895349&amp;auto_play=false&amp;hide_related=false&amp;show_comments=true&amp;show_user=true&amp;show_reposts=false&amp;visual=true"></iframe>

