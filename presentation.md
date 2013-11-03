Hi,

I'm here to talk about responsive images, both present techniques and
future solutions.

You may be asking yourself, who is this guy?
Well I've been working with on responsive images for the last 2 years,
trying to find a standard solution and reach concensus around it.
During that work, I prototyped the picture element in WebKit & Blink,
participated in the srcset implementation in WebKit and rewrote that
implementation for Blink. I also overviewed current JS techniques to
understand what's do developers need from a responsive image solution,
and what are the tradeoffs of current techniques, to understand how can
a native solution do better.

So, I'm here to share what I've learned during that roller coaster
journey :)

So let's start with the basics. How many of you know what the responsive
problem is? (show of hands) 

Why is the responsive images subject so close to my heart?

When RWD first became a thing, image handling was simply "let the
browser downsize the largest image you'd need and you'll be fine".

But soon, people realised that responsive Web sites are sending way too
much data and in particular way too much image data to mobile browsers.

Let's take a look at *how much* extra data is being sent:
The latest data from Guy Podjarny shows that 72% of responsive Web sites
send exactly the same amount of data to all viewports.
A site survey by Tim Kadlec (Using a utility that I've built, called
Sizer Soze) showed that 72% of image data can be saved when sent to the smallest viewports.

The utility he used, is something I built one Saturday morning after
reading one of Jason Grigsby's excellent posts talking about responsive
image and a performance budget. That post got me thinking that
basically, we don't know how much data we're wasting on non-responsive
images, and you can't handle a budget without accounting for your
expenses.
So I hacked a small script together that uses PhantomJS to get the
display dimensions of the page's various images, then downloads them and
resizes them to see how much we're wasting.

Let's take a look on what it does. Those of you who are interested and
have a unix based system can go to github.com/yoavweiss/sizer-soze,
clone it and install it with the either the ubuntu or OSX scripts.

[Show sizer soze on my machine. ask the audience for 5 URLs. Look at
results. Show how to analyze them]

This project is very much in the "pet project" phase. There can be bugs,
and if you guys bump into anything, don't hesitate to open a bug on the
repo.

There were also some efforts to set-up a Web front-end, and there's
even a design and everything, but currently it is kinda on hold. I hope
I'd have time in the next few-months to get it off the ground.

So, we've seen that in term of byte size, there are tons of savings to
be made, but is that all there is to it? As some people say, bandwidth
is cheap...

# Decoding speed
Well, going back to Tim Kadlec again (honestly, that dude is
everywhere), he ran a followup study that shows that decoding speed in
the browser (and Chrome specifically) is
about 10 times worse if the image is x6 bigger. But, that's not the
worst part. Resizing the images was x50 times slower than their
decoding.
So basically, if you're sending x6 images to your users, you're making
image handling about x60 times slower !!!!!
On desktops, this is something that machine can handle, but on phones,
that's a significant slow down.
Now, some browsers do that kind of work off the main thread, so the
rendering slow down might be less noticeable, but other don't yet
(firefox) and others never will (Android WebKit, and other older
browsers :( )
So, sending large images would mean burdening these users, which are
already having a rough time, with even more jank.

# Hack time
OK, so we want to load smaller images on smaller devices, how hard can
it be???
We'll just use javascript!

This is what a lot of people thought when first starting to explore the
responsive images problem. But they soon discovered they're in for a
rough ride, because of a browser optimization mechanism that was little
known back then.

# Who's afraid of the big, bad preloader???
[ Raise of hand, who knows what the preloader is? ]
It comes with many names (mainly because it's not a standard feature, so
each browser named it differently). So PreloadScanner, speculative
parser, lookahead parser and possibly more.
I'll try to explain what he does by explaining how resources are loaded
in its absence.
Let's looks at a page load with a preloader disabled.
At first, the HTML is requested and starts downloading.
Then, in one of the first packets coming in, the HTML parser finds a
blocking JS resource. 
Now, since a blocking JS resource has to run with the DOM at exactly the
state where the DOM was when the JS started running. The JS can also run
document.write calls which must be inserted into the DOM again at the
same place. Otherwise, content would break.
So, the HTML parser stops it's parsing, waiting for the JS resource to
download, parse and execute.
Only then can the parser continue it's parsing, only to find another
blocking JS resource, and repeat the same little dance.
That results in waterfall charts that look like this [Need to get a no
preloader waterfall }

So when the first responsive designs tried to make their images
resonsive as well, the basic idea was to add some JS to the head of the
page that would add some cookie to the page indicating the device's
dimensions, or swap out the base tag, so that all images would be
requested from a certain directory according to the device dimensions.

But, because of the preloader, that didn't quite work as they expected.
Since they added some JS to the page's top, the parser stopped, but the
preloader continued scanning the page and sending requests. 
The JS setting the cookie or swapping the base tag was racing against
the preloader, since in order to be effective, the JS had to run before
the requests for images are sent. And in many cases, the JS lost that
race.

That caused a lot of frustration and "hate" in the dev community to the
preloader. It became some mythical creation, taking the blame for all
dev troubles.
What we need to keep in mind is that it enables a 20% performance
improvement. 

Anyway, if we're looking at the JS solutions we have today to the
responsive images problem, we can divide them into several categories:
* LQIP
* Server side - cookie based
* Client-side picturefill like
* Web component based
* Capturing
* Bandwidth testing

Let's talk about each one of those in detail.

#LQIP
The term was coined by Guy Podjarny from Akamai. It basically means that
the img's src attribute includes a low quality image that is picked up
by the preloader and is loaded as fast as possible.
So we're taking a double download hit here, but since the first image is
of extremely low quality, the bandwidth damage is not that bad.
Further down the page, either synchrounousely or on DOMContentLoaded,
you run some JS that scans your DOM for these images, and swap the original low quality
image with the proper image according to viewport dimensions, the image
dimensions if layout took place, or anything else.
Now the UX here is very similar to the one of progressive JPEG. A low
quality image loads first, and the HQ image arrives later.

(Show an example page that uses it and go over the basic code ) 

# Server side cookie based
One of the first approaches, mainly used by Matt Wilcox's
AdaptiveImages. It uses an inline JS at the page's top to set a cookie
with the viewport dimensions, and uses a server-side backend to serve
the appropriate images.
The downside of this approach is the race condition we talked about
earlier. It rarely (if ever) works on the first page load.
Since several studies have shown that 50% of users are first time
visitors, depending on your site, that may be a problem.
It also requires server side logic that deduces image dimensions from
viewport dimensions, but this is a common concern for solutions that
don't delay the load of the actual image.

(Show an example page that uses it and go over the basic code ) 

# Client-side picturefill like
This approach gets rid of img src completely to bypass the
preloader for images. The reason for that is the desire to avoid the
double download, to avoid wasting bandwidth. That's better for the
overall age load time, but the downside is that it may
take longer until the users actually see any image data.
One of the main frameworks that use this approach is picturefill, a
polyfill of the proposed picture element.

(Show an example page that uses it and go over the basic code ) 

# Web component based
This approach is very similar to the previous one, but instead of
defining some markup pattern using spans and divs, we're defining a new,
custom element that defines that markup.
Advantages are that the markup is cleaner and in supporting browsers,
performance can be better, since the image download can be kicked off as
soon as the image is parsed, rather than waiting for DomContentLoaded.
In theory that perf benefit can also happen without native Web component
approach, but it requires adding a <script> tag after each image, which
may have it's own performance overhead, and is very cumbersome.

The prominent example is x-picture from NationalGeograhic, which was
adopted by the RICG

(Show an example page that uses it and go over the basic code ) 

# Capturing
This magnificent hack from mobify.js has some serious downsides, but
it's just too beutiful to pass by :) Brace yourselves...

Basically, they add a small inlineJS to the documents's head that
document.writes a "<plaintext>" tag. From now one, the preloader and the
parser treat the rest of the page as text, and just shut down.
Then they query the DOM, extract the original HTML, scan it for
resources and download them in parallel (???), inject the HTML back using JS.

The big advantage of that aproach is that for legacy sites that
can't/won't touch their front-end code, it provides responsive images
without double downloads. The downside is that it has a performance
penalty, and large parts of that are reimplementing the browser in JS,
which is not a winning strategy.
They claim that in term of performance, they're not losing much, since
they save tons on the image resizing.

(Show an example page that uses it and go over the basic code ) 

# Bandwidth testing
Each one of the above approaches can use some some of bandwidth testing
in order to make its decision. 
There are a couple of different approaches to bandwidth measurements:
* active download measurements, so a download of a certain resource using JS,
measuring the time it took to arrive, and deducing the bandwidth from
that. That usually introduces a further delay since basically we're
adding a blocking, unnecessary resource. We're downloading more than we
should only for the sake of measurement.
* Passive download measurements, so measurement of the download of one
  of the required resource. That can be done smarter on some browsers,
by using the navigation timing API (if we're measuring based on the
HTML), or the resource timing API, if we're measuring based on one of
the other resources.
* Network info API - irrelevant. Don't use it.

Prime examples are foresight.js and hidpi???

(Show an example page that uses it and go over the basic code ) 

* Other interesting techniques
art-direction using a CSS BG image!!!
Clown car or SVG MQs - hacky, but can serve a certain need

# Let's look into the FUTURE
The RICG & the standard bodies have been working for 2 years on various
proposals. This begining of that wasn't pretty. Lot's of fighting (I
wasn't really part of that, I joined the RICG a little later).
A few proposals
Include the Edge conf presentation here and expand on it!!!

Steve souders say - 8-10 blocking scripts per site
If the non-resp images are visible to the preloader, it's better to put
the resonsive ones on a different domain.

