---
description: >-
  A while ago I decided that I wanted to figure out what went into making a
  modern website.
---

# My First Website

_Originally Published: Apr 11 2012_

In September of last year I decided I wanted to learn how to build a website from scratch. My goal in this project wasn’t to become a programmer or designer. I just wanted to give myself a crash course in what makes a modern website work. The final product is really simple, but it ended up hitting on most of the points I wanted to touch on.

Now that the [site is live](http://www.photosandtext.com) \(not to be confused with finished\), I wanted to write up something real quick on what I learned throughout this process.

#### How the site works \(in brief\) <a id="how-the-site-works-in-brief-"></a>

When a photo is uploaded to the site, all of the thumbnails I need are created and pushed into a content delivery network, some of the EXIF data \(shutter, aperture and date taken\) is pulled from the photo and pushed into the database along with the comment, name, thumbnail urls, etc.Then as you browse the site, this data is pulled from the database, code is run and everything is assembled into a webpage for you.

#### Even the simple things are hard <a id="even-the-simple-things-are-hard"></a>

I ended up settling on creating the site using [Django](https://www.djangoproject.com/). There were several reasons I went this route, mainly because it gave me an excuse to learn Python \(which I’ve wanted to do for a long time\) and Django is a really popular web framework with plenty of newbie friendly tutorials. That came in handy as a I had to to implement a lot things I had taken for granted running wordpress. In addition to the obvious “How do I store and then display information about my photos”, I had to find out:

* How do you actually generate an RSS feed?
* How do you alter files that are uploaded? \(Creating thumbnails, deleting them when the entry is deleted from the database..\)
* How do you have a user fill out a form and then do something meaningful with it? \(Not relevant to Photos and Text, but to the RSVP site I’m building\)

These are all things that I had never given a second thought to when playing on the internet. And while Django does provide a healthy amount of abstraction between the actual database queries and some of the code generation, it was still an eye opening experience to have to build these from a much more base level than I’ve done before.

#### Facebook is a pain in the ass <a id="facebook-is-a-pain-in-the-ass"></a>

Turns out there’s a bit of work involved in getting Facebook to pick out images and such from a website. They also use their own format for all of it, so you end up having to do a lot of work twice.

[http://stackoverflow.com/questions/2987195/how-does-facebook-know-what-image-to-parse-out-of-an-article](http://stackoverflow.com/questions/2987195/how-does-facebook-know-what-image-to-parse-out-of-an-article)

#### It’s easy to say stuff. It’s a lot harder to explain it to a computer. <a id="it-s-easy-to-say-stuff-it-s-a-lot-harder-to-explain-it-to-a-computer-"></a>

I call the 15 photos underneath the main photo, the filmstrip \(cause I’m clever\). The idea was pretty simple:

For a given photo, display the previous 7 and next 7 photos If the photo only has 6 previous photos, display those photos and the next 8 This should work in reverse too \(4 next, previous 10\)

Actually implementing this turned out to be the most frustrating part of the entire site. I finally sat down for three hours one Saturday and just pounded my head on it until I figured it out. There are probably simpler ways to have accomplished this, but what I ended up doing was this:

```text
#photo = The current photo.
#Seed = the 15 photos previous to (and including) the current photo
seed = Photo.objects.filter(id__lte=(photo.id)).order_by('-date_posted')[:15]

#If there are less than 7 photos next, count = 14 - how many photos are next
if Photo.objects.filter(id__gt=photo.id).count() < 7:
    count = (14 - Photo.objects.filter(id__gt=photo.id).count())

#Else, if there are less than 7 photos previous, count = how ever many photos we grabbed 
#at the beginning - 1. The minus one allows us to use count for our position later.
elif Photo.objects.filter(id__lt=photo.id).count() < 7:
    count = seed.count()-1

#If none of the above matches, count = 7
else:
    count = 7

#Here’s the magic. This grabs 15 photos, starting from a position in the initial seed 
#list, determined by what our count variable ended up being.
pstrip = Photo.objects.filter(id__gte=(seed[count].id))[:15]
```

Code for PhotosandText is avalable [here](https://github.com/jaredhaight/pat_site)

