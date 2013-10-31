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

Let's start with the basics? Why bother?

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
That results in waterfall charts that looks like this [Need to get a no
preloader waterfall }




Tell the whole javascript preloader thing!!!





art-directions using a CSS BG image!!!

Bandwidth detection is crap!

Clown car or SVG MQs - hacky, but can serve a certain need
