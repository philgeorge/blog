Title: IE6 Layout Fixing
Published: 01/06/12
Tags:
  - browser
  - html
  - css
---
![Broken IE6 logo](/posts/img/ie-broken.png){align="right"}
In the temporary absence of our resident HTML expert, today I took upon myself the often thankless task of trying to fix a number of web page layout issues where only IE6 is affected.
Our product, the [Global Corporate Challenge](https://www.virginpulse.com/globalchallenge), appeals to medium and large corporate customers,
who for one reason or another have been unable to update their employees' PCs.
This  means that we have a significant number of users who are still using Windows XP with IE6.

<small>(Thanks to [SoftIcons](http://www.softicons.com/application-icons/psferox-icons-by-psferox/internet-explorer-icon) for the image.)</small>

Here's a list of the techniques I used to tackle this challenge:

- Most commonly things were appearing more spaced out in IE6. This is usually caused by the infamous IE6 doubled float margin bug. The fix is to add the style `"display:inline"` on any floated elements that are out of position.
- An empty div used just to add a background image was creating a full line height of blank space in IE6. The fix was to set `"font-size:1px"` on this div.
- Some floated label elements were completely invisible. In this case, adding `"position:relative"` brought them back to life.
- Change `"background-color:#ffff32"` styles to `"background:#ffff32"` because IE6 doesn't support the more specific style name.
- We have been using the clearfix technique in much of our layout but there were a few areas of content where it hadn't been applied. So this was simply a case of changing
```
                <div>stuff</div>
                <div class="clear"></div>
```
to
```
                <div class="clearfix">stuff</div>
```

So satisyingly most of the issues were fixed. Still, I continue to long for the day when we no longer need to support this 12-year-old browser!
