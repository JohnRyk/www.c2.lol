---
description: >-
  I built a CMS for my brother and friends. Here's a look back at eight months
  worth of work.
---

# Building Live Geek or Die

_Originally Published: April 20th, 2013_

Occasionally I get the bug to build something. Normally when that happens I don't actually have any ideas as to what I want to build. Which leads to facebook posts like this:

![](https://www.psattack.com/webhook-uploads/1448120698853/20120815_facebook_post.png)

Shortly after posting that, my brother reached out to me and let me know that he and his friends were looking to update their website, Live Geek or Die. Live Geek is a gaming/tech news/blog. While I think their initial thoughts were just a theme change, the idea of building a Content Managment System \(CMS\) was exactly the sort of over reaching project I was looking to take on.

Eight months later, we launched the new [Live Geek](http://www.livegeekordie.com).

#### Making it look good <a id="making-it-look-good"></a>

With so much web browsing coming from mobile and tablets, I don't understand why anyone would design a site these days that isn't responsive. For Live Geek, [Bootstrap](http://twitter.github.io/bootstrap/) handles much of the "responsive grid", which is what lets the various chunks of the page collapse down on smaller screens. For example, on the desktop page, there are two columns of articles and on mobile this collapes down to one column.

![](https://www.psattack.com/webhook-uploads/1448120726253/20130420_lgod_home.png)

The home screen at different sizes.

Helping Bootstrap, is a javascript library called [ResponseJS](http://responsejs.com/). ResponseJS swaps in html depending on what size the screen on the browser is. I'm using it to swap in larger images for larger screens. The image size for any article image on the site is 1500px wide and 400px tall \(that's really wide\). We don't want to serve a 1500px wide picture to a phone on a 3G connection, so when an image is uploaded to the site I generate a series of crops \(650x400, 800x400 for headers as well as various other sized cropped/resized versions for other areas of the site\). We then take those crops, and swap them into the page depending on what size the screen is.

By default, we serve out the smallest image. That means that we don't waste precious bandwidth for mobile devices. If you connect on your phone, you're going to get smaller images. If you connect from your desktop, you may start to get smaller images, but they'll swap over to the larger versions automatically \(and generally transparently\). This keeps page size down as much as possible.

Between Bootstrap and ResponseJS, the site loads quickly and looks great no matter what device you're using.

#### Making it quick <a id="making-it-quick"></a>

The other thing that I was very concerned about on the frontend was speed. My goal was to get the home page to load in under two seconds. Obviously, there's a billion variables that are going to swing load time one way or the other, but if I could see regular &lt; 2 second loads, I would be happy. In the end, I'm pretty damn happy.

![](https://www.psattack.com/webhook-uploads/1448120749838/20130420_lgod_load.png) 

To achieve this, I didn't do anything particularly clever:

* [Varnish](https://www.varnish-cache.org/) up front, caching pages cooked up from Django
* Server the smallest possible versions of the images on the page, depending on screensize \(see ResponseJS above\)
* [Cloudfiles](http://www.rackspace.com/cloud/files/) is used to serve up static assets and images. This serves two purposes, CDNs are great and it allows more simultaneous requests to the main live geek server.
* Images are lazy loaded \(meaning that images are only loaded when they're about to be show on screen\)

#### Building Tools <a id="building-tools"></a>

Most of the work on the site was spent on stuff only a handful of people would see.

**Article Editor**

The article editor has a serveral cool features. Like the rest of the site, it's responsive. If the guys want to post an article from their phone it's totally doable.

![](https://www.psattack.com/webhook-uploads/1448120802070/20130420_lgod_editor.png)

The editor at different sizes

Because I wanted them to be able to post for their phones, I also built up an autosaving system. On the page, javascript runs every five seconds and takes the values from the fields in the editor, converts them to JSON and then sends them back to the site. If the save is successful, a little disk flashes on screen:

```text
/*
Grab values from fields, convert to json and post back to 
article autosave URL
*/
function autosave(){
    var title = $('#id_title').val();
    var summary = $('#editorSummary').val();
    var categories = $('#editorCategories').val();
    var type = $('#id_type').val();
    var body = $('#editorBody').getCode();
    var dataString = {'title':title, 'summary':summary,'categories':categories,'body':body, 'type':type};
    var jsonText = JSON.stringify(dataString);
    $.ajax({
        type: 'POST',
        url: '/editor/{ article.id }/autosave/',
        data: jsonText,
        success: saved // call "saved" function if successful
    });
}

//Flash the little disk icon on screen and fade it out
function saved(){
    $('#editorSaved').show();
    $('#editorSaved').delay(500).fadeOut('slow');
}
```

The backend takes the JSON sent over from the javascript and then updates the article. I'm sure there are a whole bunch of ways I could make this more efficient \(rather than a series of try and if statements\), but it works so that's good.

```text
def articleAutosave(request, article_id):
    #Initialize status variable as an empty dictionary. 
    #We're going to append values to status depending on 
    #what get's saved later. We also grab the article object
    #so we have something to save our values against.
    status = []
    article = Article.objects.get(pk=article_id) 

    #Here we load the json that the javascript sent to a 
    #variable (data) and then check to see if has the bits 
    #we're looking for (body, title, summary, etc)
    try:
        data = json.loads(request.body)
    except:
        data = None
    try:
        title = data['title']
    except:
        title = None
    try:
        body = data['body']
    except:
        body = None
    try:
        summary = data['summary']
    except:
        summary = None
    try:
        type = data['type']
    except:
        type = None
    try:
        categories = data['categories']
    except:
        categories = None

    #For each item, we check if it exists. If it does, we
    #add it to the article and load our response ('item':'saved')
    if title:
        article.title = title
        status.append({'title':'saved'}) 
    if body:
        article.body = body
        status.append({'body':'saved'})
    if summary:
        article.summary = summary
        status.append({'summary':'saved'})
    if categories:
        article.categories = categories
        status.append({'categories':'saved'})
    if type:
        article.type = type
        status.append({'type':'saved'})

    #Save the article and return our response of what was saved
    article.save()
    return HttpResponse(json.dumps(status), mimetype="application/json")
```

**Staff Dashboard**

The other cool thing I did was created a dashboard where people could see how their articles are doing. I wrote a service that checks for Twitter shares, Facebook likes and comments and page views out of Google Analytics for the articles on the site. The service runs every 15 minutes and gets the data for the articles posted on the site within the last 30 days. Articles shown on the dashboard are limited based on user permissions where normal staff can see their articles and editors can see all articles.

![](https://www.psattack.com/webhook-uploads/1448120843008/20130420_lgod_social.png) 

Surprisingly, Facebook and Twitter were really easy to work with. Basically just a query to a URL. Getting Pageviews out of Google Analytics required a lot of reading and then copying and pasting code from Stack Overflow and hoping for the best. One of these days I'll learn how to OAuth.

```text
#The "service" here is the end result of authenticating to 
#Google via OAuth. Here we pass in that service as well as
#the article that we want pageviews for.
def get_google_stats(service, article):
    date_posted = article.date_posted.strftime('%Y-%m-%d')
    today = (datetime.date.today() + datetime.timedelta(days=1)).strftime('%Y-%m-%d')
    url = 'ga:pagePath==/article/'+article.title_slug+'/'
    # Use the Analytics Service Object to query the Core Reporting API
    return service.data().ga().get(
        ids='ga:' + GA_PROFILE_ID,
        start_date= date_posted,
        end_date=today,
        metrics='ga:pageviews',
        filters=url).execute()

#The end result of "get_google_stats" is a lot of data.
#This function takes that data and returns what we want
def get_pageviews(google_stats):
    # Print data nicely for the user.
    if google_stats.get('rows'):
        return google_stats.get('rows')[0][0]
    else:
        return 0

#Getting facebook stats is as simple as sending a request
#to "http://graph.facebook.com/{url}", where "{url}" is the
#site you want. For example: http://graph.facebook.com/http://www.google.com
#This returns JSON that you can parse really easily.
def get_facebook_shares(article):
    url = BASE_URL+'article/'+article.title_slug+'/'
    r = requests.get('http://graph.facebook.com/'+url)
    try: shares = r.json['shares']
    except: shares = 0
    return shares

#Same for twitter. Request to 
#http://urls.api.twitter.com/1/urls/count.json?url={ url }
def get_twitter_stats(article):
    url = BASE_URL+'article/'+article.title_slug+'/'
    r = requests.get('http://urls.api.twitter.com/1/urls/count.json?url='+url)
    try: tweets = r.json['count']
    except: tweets = 0
    return tweets

#This is like, the uber function. Feed it an article and
#it will run all the social functions listed above and then
#save the results.
def update_social_stats(article):
    stats = get_google_stats(service, article)
    article.social.pageviews = get_pageviews(stats)
    article.social.tweets = get_twitter_stats(article)
    article.social.facebook = get_facebook_shares(article)
    article.social.save()
```

#### Solving Problems <a id="solving-problems"></a>

The biggest thing I learned about from this project is Context Processors in Django. I have a lot more reading to do on the subject, but my understanding is that Context Processors let you work with a request on the site before it makes it to your templates \(templates being the HTML for Django\). In my case, I'm checking the request for a couple of attributes and passing variables to the templates based on those attributes.

Let me give an example. All of this came about when I tried to load the site over HTTPS. The CSS and Javascript for the site are coming from `http://files.livegeekordie.com` \(which is really just pointed the CloudFiles CDN\). Browsers do not like running javascript from http if you're supposed to be on a secure site \(https\). To resolve this, I wrote a context processor that checks to see if the request is not secure \(http\), if it is it makes sure that the variable to load assets from is `http://files.livegeekordie.com`. If the request is secure \(https\) then the variable is left as an https link to the CDN \(I defaulted to https because it's the safest option\)

```text
def manage_static_url(request):
    if not request.is_secure():
        return { 'STATIC_URL': NONSECURE_STATIC_URL }
    else:
        return { 'STATIC_URL': STATIC_URL}
```

Overall, building the Live Geek was a great expierence that brought up a lot of interesting challenges and taught me a lot about Django. It also did a good job of teaching me what I have to work on, I think largest of which is writing code that is less monolithic.

Code for Live Geek or Die is available [here](https://github.com/photosandtext/lgod/).

