---
layout: post
title:  "Creating a simple EPiServer plugin"
date:   2019-06-21
---

One of my main projects is a site that was built on the EPiServer CMS a couple years ago. The site depends on a set of product data, and I wanted to give site admins the ability to manage that data in the EPiServer admin interface. I knew EPiServer could be extended with plugins, but I had a difficult time figuring out the exact steps to make my own plugin. 

Eventually I discovered that making a basic plugin is actually really simple. It only requires a controller and a view, plus two more key pieces:

//controller and view - authorized
//menu provider