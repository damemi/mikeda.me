---
title: "My First Simple Chrome Extension"
date: 2017-12-04T02:39:06
draft: false
slug: first-simple-chrome-extension
categories:
  - Uncategorized
tags:
  - brewing
  - chrome
  - extension
  - javascript
wordpress_id: 146
---
Homebrewing is one of my favorite hobbies, so I'm a big fan of [Brewer's Friend](https://www.brewersfriend.com/) for looking up recipes (as well as using their various calculators for OG/FG, IBUs, and others). Their search functionality is very good, and makes it easy to find exactly the kind of recipe you're looking for.

What's odd, however, is that on their recipe pages they will show a total amount of grain used:

![grain](http://www.mikeda.me/wp-content/uploads/2017/12/grain.png)

But for hops, they'll only show the total for *each type* of hop:

![hopsold](http://www.mikeda.me/wp-content/uploads/2017/12/hopsold.png)

It's a minor nitpick, but I wanted to also see the total amount of hops in a recipe. So, I figured it was finally time to get around to dabbling in Chrome Web Extensions. So following [this tutorial](https://carl-topham.com/theblog/post/creating-chrome-extension-uses-jquery-manipulate-dom-page/), I dug right in. My manifest.js  file came out like this:
 {
  "name": "Brewer's Friend Hop Total",
  "version": "1",
  "manifest_version": 2,
  "description": "Shows the total amount of hops in a recipe",
  "background": {"page": "background.html"},
  "browser_action": {
    "name": "HopTotal",
    "icons": ["icon.png"],
    "default_icon": "icon.png"
  },
  "content_scripts": [ {
    "js": [ "jquery.min.js", "background.js" ],
    "matches": [ "http://www.brewersfriend.com/homebrew/recipe/view/*", "https://www.brewersfriend.com/homebrew/recipe/view/*"]
  }]
}
Pretty straightforward: It lays out the title and description of my extension while also including some necessary versioning metadata. It also includes denotes which pages I want the extension to run on and includes my javascript, which I hacked together to look like this:
total = 0
$('#hopsSummary .brewpartitems td').each(function() {
     value = $(this).text();
     if(value.indexOf('oz') >= 0) {
          len = value.indexOf(' oz');
          val = value.substr(0, len);
          total = total + parseFloat(val);
          console.log(val);
          }
     });
console.log(total);
$('#hopsSummary table tr:last').after('  **'+total+' oz**   **Total**   &nbsp;   &nbsp;   &nbsp;  ');
I found the table classes and IDs by poking around the Brewer's Edge source. This just loops through each cell in the "Hops Summary" table looking for cells that list a number of ounces ("oz") and strips out the number, summing as it goes. I then just add a table row (I copied that html from the grain total row above).

I loaded the extension into Chrome by going to Menu > More Tools > Extensions and checking the "Developer mode" box. Then I just had to click "Load unpacked extensions..." and navigate to the folder with my code in it. Success!

![success](http://www.mikeda.me/wp-content/uploads/2017/12/success.png)

Now I wanted to figure out how to get it onto the Chrome store. This step was more involved than I expected, first off I had to find the [Chrome Developer Dashboard](https://chrome.google.com/webstore/developer/dashboard) and clicked "Add New Item". This form was pretty straightforward, but remember you need to make a 250x250 icon, a screenshot, and at least a 440x280 promotional tile. You also need to verify that it belongs to a website you own, which is a process I didn't know about with Google. Basically what it amounted to was clicking on "Add a new site" and adding a TXT entry to my sites DNS (this step held me up because I was adding the DNS entry for the "www" host when it should've been "@").

After that just fill out some categorical/analytics info and click publish! It took about 20 minutes for my extension to show up in the Chrome Web Store. [Here's a link to the Brewer's Friend Hop Total extension.](https://chrome.google.com/webstore/detail/brewers-friend-hop-total/glfiafplgekdihjbpinefckiienkojil)

**Update:** After I published this extension, I shared it on [r/homebrewing](http://reddit.com/r/homebrewing) and went to bed. I woke up the next day to see it on the front page of the subreddit, with 80+ upvotes! (Not bad for there). By the end of the day Brewer's Friend had added the functionality to their website, ultimately rendering my extension useless in what I suppose you could call a successful exit. An exciting day for sure, and a good cap on my first Chrome extension.

![store](http://www.mikeda.me/wp-content/uploads/2017/12/store.png)

*Note: This extension is unofficial and in no way affiliated with Brewer's Friend*
