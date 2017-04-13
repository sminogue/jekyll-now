---
layout: post
title: Simple Document Scanner in Java
---

So recently in my travels I came across [Adrian Rosebrock's blog](http://www.pyimagesearch.com/), hes doing a lot of really interesting Computer Vision applications which I have enjoyed reading about. One of his posts was about creating a [simple document scanner in 5 minutes](http://www.pyimagesearch.com/2014/09/01/build-kick-ass-mobile-document-scanner-just-5-minutes/) caught my eye and captured my imagination. I had to tinker a bit as the post is written using openCV2 and the current is openCV3 which has some differences.

Being a Java developer I thought it would be fun to do a similar application in Java rather than python. I also didnt want to use openCV for my application. So here is my version of the quick (not mobile) document scanner in pure Java.

To make this work and not use openCV I first needed to find a good Java computer vision library. After searching around a bit and discarding some Java wrappers for openCV I came across [boofCV](https://boofcv.org/) Ive included the XML from my POM to help you find it. So with that in hand lets get started!

{% gist 1e4f3d46ff130605ae4e45f1ffae37cd %}

So first off I created a simple maven project (its just easier for me to do dependency management with it but feel free to use whatever you like). I then created a simple class with a main method that will do the work and proove the concept. I also chose to use some visualization classes provided by boofcv.

So to get started I added the following to my main method:

{% gist 0338aa50b3c24d95f9b28e69e3afcc94 %}

When running this you should be presented with a screen that displays nothing. So now we have a running application which is ready to display images. So lets load a test image and do some prep work on it to prepare page detection.

{% gist e5e43d2d13a2f8ab312adbf9fbf3f797 %}

Now that the image has been loaded and we have done a little prep work we can detect the page.

{% gist 473911bc6efca0f89a9f54f762b00b09 %}

Ok great so at this point with have enhanced the image and used a detector to find a 4 sided polygon (the page). Now we just need to identify the 4 corners of the polygon an put them in TL,TR,BL,BR order. This is necessary as the boofcv library we will use to transform the image needs the points in that specific order. The transform is going to change the perspective to be looking straight at the page and the page be straightened out. This way if the picture is not PERFECTLY aligned we still get a good scan. I wrote a little method which will sort a list of points into the required order. Its in the first code block below. the second contains how we use it.

{% gist 751c837f516572f62a10e1dc75bac667 %}

{% gist 4400e46a614989a3fabe17c880a22397 %}

So at this point we now have the four corners identified and we can transform the image.

{% gist 7a44ae666df5047b1489047f376a6313 %}

At this point we have the image cropped and the perspective shifted to be straight. The last thing to do is to convert the final image to black and white and apply an adaptive threshold to remove shadows or any kind of discoloration.

{% gist 5435bfb45a11498b7cccf5625bdd1a93 %}

And there it is! If everything has worked properly we should be left with just the document in the picture and it should be a nice clean black and white.

Complete source for this can be found as a [gist on github](https://gist.github.com/sminogue/7bd865e9e46a9a2ab079c22af08d1c0d "gist on github").

