Title: Windows Phone 7 Cannot Render ASP.NET page
Published: 7 June 2013
Tags:
  - ASP.NET
---

![Generic smartphone in hands](/posts/img/smartphone-in-hands.jpg){align="right"}
We had been running our ASP.NET MVC site for years with over a hundred thousand happy users until one day recently our team's boss decided to break from the iOS/Android norm and got himself a new Windows Phone 7. To everyone's astonishment, when visiting [our site](www.gettheworldmoving.com) in the WP7 default browser (IE9), it rendered nothing but the title tag, and even that was just as plain text.


Some adjustments to our main layout page caused other tags to be rendered, but never as proper HTML.

Eventually, on a hunch, I took a closer look at our App_Browsers folder. It had been in our ASP.NET project for about 2 years (from the original default Visual Studio template files) and never updated.

The ASP.NET browser identification system was not correctly identifying this WP7/IE9 combination and applying a response header of `content-type:"application/xhtml+xml"`. The mobile IE9 browser clearly didn't know how to handle this and simply displayed the start of the file as if it was XML.

One solution would have been to update our App_Browsers files, but I considered this an unecessary overhead on our team. Instead I opted to simply remove App_Browsers and everything within. We are building our site always serving hand-written HTML5 text/html, using media queries to respond to smaller screen sizes. This approach doesn't any special server-side browser detection. And it works just fine on the boss' Windows Phone 7!
