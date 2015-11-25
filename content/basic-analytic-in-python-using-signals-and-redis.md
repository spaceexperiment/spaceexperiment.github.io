Title: Basic analytic in Python using signals and Redis
Date: 2014-12-02 19:30
Author: jibreel
Category: python
Slug: basic-analytic-in-python-using-signals-and-redis

A slightly underused(or less needed for most projects) feature of
frameworks are the signals they provide.

Signals allows you to hook a certain action initiated to some code that
you provide.

For example, whenever there is a request comming to your web
application, then your framework initiate a signal at the begining of
the request (before it reaches your view) and a signal at the end of the
request (after you have rendered your view and it's ready to be sent to
the user).

Here is an examples for Django and Flask on how you could use them to
make basic analytics of your site.

<span class="highlight-blue">Flask</span>

```
app = Flask('my_app')

@app.before_request
def analytics():
    if request.path.endswith('/'):
        redis.zincrby('visitors', request.path)

    redis.zincrby('referrers', request.referrer)
    redis.zincrby('browser', request.user_agent.browser)
    redis.zincrby('platform', request.user_agent.platform)
```

<span class="highlight-blue">Django</span>

```
# We will be using middleware.

class Analytics(object):
    def process_request(self, request):
        if request.path.endswith('/'):
            redis.zincrby('visitors', request.path)

        redis.zincrby('referrers', request.META.get('HTTP_REFERER', ''))
        redis.zincrby('user_agent', request.META.get('HTTP_USER_AGENT', ''))
```

Although this is a very simple example, but they don't have to be overy
complex to be usefull.

This is taking data from the request and storing them in redis as
ordered set.

Every time the same value occures it will increase it's score by one,
that's what `zincrby` does.

The first one is the url path, this allows you to track how many page
views any page had.

Note that we are adding it only if the url ends with "/" because usually
if someone visites `http://.../path` they
get redirected to `http://.../path/` with a
slash at the end, so we don't save it twice.

The second one is for the referers, when a user get to your page through
another page the browser would set the Referer header in the HTTP
request so you know where it is coming from.

But now that this can easy be forged, any one can visit your site and
modify this header by intention.

The third is the browser the user is using, the framework works this out
from the HTTP header `User-Agent`.

The fourth is the platform (linux, windows, ios ...), this is also
worked out from the `User-Agent` header.

See, in just a few lines of code we are accumelating data about page
views, where they are coming from. what browser they are using, and what
platform or OS.

In Django you will have to include that in as middleware e.g. save this
as mymiddleware.py in your app folder and add
'app.mymiddleware.Analytics' to the setting MIDDLEWARE\_CLASSES.

Also we used `HTTP_USER_AGENT` instead of
browser and platform, that is because django doesn't parse this header
string, you will have to use a library to parse that.

The header string looks something like this `Mozilla/5.0 (iPhone; CPU iPhone OS 5_1 like Mac OS X)
AppleWebKit/534.46 (KHTML, like Gecko) Version/5.1 Mobile/9B179
Safari/7534.48.3`

You can use [this library](https://pypi.python.org/pypi/user-agents/) to
parse it.

To read the data simply query the keys with redis, e.g.

```
>> redis.zrange('visitors', 0, -1, withscores=True)

>> [('/atom/', 2922.0), ('/blog/1/', 8360.0), ('/', 8714.0)]
```

A tuple of the path and the score(view count)

There are some other uses and signals available, for database `pre_save`,
`post_save`, `pre_delete` .. and so on.

Other uses is also by tracking the users remote ip and ignore a request
after a certain amount of POST request for a form, so for example after
posting a 5 comments in an hour then just ignore the POST request, you
don't handle it to avoid spam.

Although this is good, it's not perfect as one can use proxy, but for
most cases this would work.

