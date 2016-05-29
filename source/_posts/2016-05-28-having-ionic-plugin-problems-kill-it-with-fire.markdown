---
layout: post
title: "Having Ionic plugin problems? Kill it with fire!"
date: 2016-05-28 13:51
comments: true
categories: [Ionic, Ionic plugins, Kill it with fire]
---

Ever have one of those days when your Ionic plugins just won't work?

Running `ionic state reset` should solve most issues.

But if that doesn't work there is only one thing left to do - **kill it with fire!**

<!-- more -->

# Kill it with fire!

When `ionic state reset` isn't enough it's time to break out the big guns!

Here's my **World Famous<sup>TM</sup> Kill it with fire!** recipe:

```
rm -rf ./plugins
rm -rf ./node_modules
rm -rf ./www/lib

ionic platform rm ios
ionic platform rm android

npm install
bower install

ionic platform add ios
ionic platform add android
```

Shout out to [Ryan LaBouve](https://twitter.com/ryanlabouve) for his help nurturing and growing **Kill it with fire!** from a simple home remedy into the world class recipe you see today.