# Subtle Jekyll Theme v1.0.1

[Theme Live Demo](https://subtle.netlify.com/)

## Features

* Mobile-ready
* Multi-author ready
* Author page support
* Tag page support
* Contact form built-in 
* Social icons built-in
* Social sharing built-in
* Mailchimp subscription form
* Code Syntax Highlight with [Prism.js](https://prismjs.com/)
* Support for Disqus comments

## Getting Started

**Table of Contents**

* 1. Theme Configuration
* 2. Author configuration
* 3. Tag configuration
* 4. Contact form setings
* 5. Social Links
* 6. Site Navigation
* 7. Images
* 8. Local Installation
* 9. Deployment
* 10. Support

### 1. Theme Configuration

The theme configuration options can be found within the **_config.yml** file. More information about Jekyll configuration can be found in the Jekyll documentation.


* name - the title of your blog, shown in the page and description areas.
* description - the description of your site for social meta tag, search engines, and feed.xml.
* logo - the image for site logo.
* greetings - used in the home page for the intro section.
* favicon - the icon for your site.
* baseurl - the subpath of your site, e.g. /blog, for generating urls. If baseurl is set, you will need to prepend the baseurl to these settings: author image, site navigation, post images.
* production_url - the base hostname and protocol of your site for where absolute urls are needed.
* disqus - your Disqus shortname. Enter the Disqus shortname here if you wish to have Disqus comments enabled, leave blank to disable comments.
* mailchimp_url - your form action URL for MailChimp newsletter signup form.
* cover_image - the cover image used for you site intro section.

### 2. Author configuration

To add new author go to **_data** directory and edit authors.yml file acording to your author information. After that you have to add new author to your post front matter.

### 3. Tag configuration

To add new author go to **_data** directory and edit tags.yml file acording to your tag information. After that you have to add new tag to post front matter.

### 4. Contact form setings
To make contact form work make sure you have defined "email: youremail@email.com" in **_config.yml** file and verify your form on formspree.io.

* email - email used for contact form.
* contact_page_description: - description used in contact form page (contact.html).
* thankyou_page_description - description used in thank you page (thank-you.html).

### 5. Social Links

To enable social links on your blog simply enter your social profile username, for example, twitter: "justgoodthemes" . If a field is left blank, the social icon will not be shown.

### 6. Site Navigation

The site navigation can be found in the **_config.yml** file. To add a page to the site navigation simply add your new page in the markdown format (e.g. newpage.md) in the theme root folder. Next edit your navigation menu located in **_config.yml** file on line 26. To add a new item to the navigation you have to add the item name and url. For example:

~~~~
navigation:
- text: New Page
url: /newpage
~~~~

### 7. Images

Images for pages are located in the horace/assets/images folder and images for posts are located in the horace/assets/images/posts directory.

#### Image With Caption

Within your blog posts you can include captions for images. This requires using some HTML markup.

The example below illustrates how to include an image with a caption in a blog post:

~~~~
{% include image-caption.html imageurl="/images/posts/Apple-Watch-In-Car.jpg" 
title="Apple Super" caption="supertest" %}
~~~~

Add the following code into your post/page markdown and change its attributes accordingly.

#### Full Width Image With Caption

To have wide images in posts or pages simply add #wide word with the hashtag at the end of image path like in the example below:

~~~~
{% include image-caption.html imageurl="/images/posts/Apple-Watch-In-Car.jpg#wide" 
title="Apple" caption="This is caption" %}
~~~~

Add the following code into your post/page markdown and change its attributes accordingly.

#### Image alignment

To align images left or right just us the **div** with class **alignleft** or **alignright**. Please, check the example below:

~~~~
<div class="alignright">
    <img src="/assets/images/posts/2018/5.jpg" alt="image name">
</div>
~~~~

### 8. Local Instalation

To set up Jekyll on local machine please follow the official documentation that can be found here -> https://jekyllrb.com/docs/.

### 9. Deployment

Sites built using Jekyll can be deployed in a large number of ways due to the static nature of the generated output. Here are some of the most common ways:

#### Netlify

This theme is intended to be used with **GitHub/Gitlab + Netlify** workflow. You set up a private repository on your chosen Git service and link it to Netlify. You can find here how to do it [here](https://www.netlify.com/blog/2015/10/28/a-step-by-step-guide-jekyll-3.0-on-netlify/). As this theme is Netlify ready, you don't have to to create any Gemfile files. After you set up Git repo, just start from the step **"Connecting to Netlify"**. To read more about Netlify features visit the following this [link](https://www.netlify.com/features/). As for Netlify pricing, the basic plan is free that will covers all blogging needs.

#### Amazon S3

If you want to host your site on Amazon S3, you can do so by using the [s3_website application](https://github.com/laurilehmijoki/s3_website). It will push your site to Amazon S3 where it can be served like any web server, dynamically scaling to almost unlimited traffic.

#### Manual Deployment

Jekyll generates your static site to the **_site** directory by default. You can transfer the contents of this directory to almost any hosting provider to get your site live. Here are some manual ways of achieving this:

##### FTP

Most traditional web hosting providers let you upload files to their servers over FTP. To upload a Jekyll site to a web host using FTP, run the jekyll build command and copy the contents of the generated **_site** folder to the root folder of your hosting account. This is most likely to be the httpdocs or public_html folder on most hosting providers.


### 10. Support

The documentation included provides all the information you need to get started with the theme. However, if you have any questions you can email us at hello@justgoodthemes.com, and we will be happy to help you.

*Also, if you have any bug reports, or feature requests, please let us know!*
