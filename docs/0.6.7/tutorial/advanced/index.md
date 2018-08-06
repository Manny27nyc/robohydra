---
layout: default
---

RoboHydra server advanced tutorial
==================================

If you haven't already, please read the [beginner's
tutorial](../) first.

Chaining
--------

If you look at the filename for the usecase teaser image we have
copied into our `new-assets` folder (see the RoboHydra front page
example in the basic tutorial), you'll notice that it contains a
version number (the `v1` bit). Now say that we don't want those
version numbers in our filenames (eg. because we want our local files
to work regardless of the current version on the production front
page). For these situations, RoboHydra has a simple solution:
chaining. The idea behind chaining is that each request handler can
accept an extra parameter `next`, a function that allows the head to
call any heads below it. As that function can be called with any
request and response objects, you can do interesting things like
tweaking the request being processed (eg. change the URL, add or
remove headers, etc.) or tweaking the response being returned
(eg. change the body or the headers). You could even call the function
several times, to retry a request, combine the responses of several
requests, or whatever else you might need.

In this example, we're simply going to strip the `.vXXX` part of the
filenames inside `/static/img/`, where `XXX` are numbers. To do so,
simply add a new head at the top and tweak the `documentRoot` in the
second head:

{% highlight javascript %}
var RoboHydraHeadFilesystem = require("robohydra").heads.RoboHydraHeadFilesystem,
    RoboHydraHeadProxy      = require("robohydra").heads.RoboHydraHeadProxy,
    RoboHydraHead           = require("robohydra").heads.RoboHydraHead;

exports.getBodyParts = function(conf) {
    return {
        heads: [
            new RoboHydraHead({
                path: '/static/img/.*',
                handler: function(req, res, next) {
                    req.url = req.url.replace(new RegExp("\\.v[0-9]+"), "");
                    next(req, res);
                }
            }),

            new RoboHydraHeadFilesystem({
                mountPath: '/static/img,
                documentRoot: 'new-assets-unversioned'
            }),

            new RoboHydraHeadProxy({
                mountPath: '/',
                proxyTo: 'http://robohydra.org'
            })
        ]
    };
};
{% endhighlight %}


Now, create a new directory `new-assets-unversioned` with the same
files, but renaming `usecases-teaser.v1.png` to
`usecases-teaser.png`. Once you have the new files, start RoboHydra
again with `robohydra -n -P rhimages`. Everything should keep working
as before, and will keep working even if the RoboHydra front page
changes the version number in the URLs.

But what about tweaking the response we get from some other head?
That's interesting, too. Let's turn instances of "server" into
"SERVER". To do that, we have to create another head before the
`RoboHydraHeadProxy`. This new head will call the proxying head with
the `next` function, but passing a fake response object. Then it will
tweak the body of that response, _then_ return that tweaked
response. It sounds complicated, but the code is simple enough:

{% highlight javascript %}
// This at the top of the file
var Response = require("robohydra").Response;

// ...

// Then, right before the proxy head...
new RoboHydraHead({
    path: '/.*',
    handler: function(req, res, next) {
        var fakeRes = new Response().
            on('end', function(evt) {
                evt.response.body =
                    evt.response.body.toString().replace(
                        new RegExp("server", "ig"),
                        "SERVER"
                    );
                res.forward(evt.response);
            });

        // Avoid compressed responses to avoid having to
        // uncompress before processing
        delete req.headers["accept-encoding"];
        next(req, fakeRes);
    }
}),
{% endhighlight %}

The `Response` constructor receives an argument, a function to be
executed when the response ends. We use this function to inspect the
response we got from the other head, tweak it and send our own
response. Also, note how we remove the "Accept-Encoding" header from
the request before proxying it: this is to avoid that the response
comes compressed. We could have also received the normal request and
uncompress it, but this is simpler for this example.

In fact, this is a relatively common thing to do, so there's a head
for specifically this purpose: `RoboHydraHeadFilter`. Using this head,
you don't have to care about compression in the server response, and
the code is much more compact and readable. A new version of the above
head using `RoboHydraHeadFilter` could be:

{% highlight javascript %}
var RoboHydraHeadFilter = require("robohydra").heads.RoboHydraHeadFilter;
// ...
new RoboHydraHeadFilter({
    path: '/',
    filter: function(body) {
        return body.toString().replace(
                        new RegExp("server", "ig"),
                        "SERVER"
                    );
    }
}),
{% endhighlight %}



Test suites
-----------

If you're serious about testing your client code, you're probably
going to end up writing a test suite of some sort. Not necessarily
completely automated, but at least you will want an easy way to change
RoboHydra's behaviour to match each test scenario.

Now, you could do that with everything we have learned up until now:
you could define many heads for the same path and enable or disable
all the relevant heads from the web interface. But that is complex and
tiring, not to mention very error prone. Alas, RoboHydra has a much
better way to deal with that situation: defining scenarios. A scenario
consists of a collection of heads. These heads are active only when
the scenario they belong to is active (and only one scenario can be
active at any given time).

Thus, if you need to easily restore a certain behaviour in RoboHydra,
you can define, in a named scenario, a collection of heads that define
that behaviour. Then, every time you need to restore that behaviour,
you start the corresponding scenario.

Let's go back to the first example in the first part of the
tutorial. In it, we made RoboHydra return certain "search results" in
the `/foo` path. If we're serious about testing that client, we
probably want RoboHydra to return different things, like no results, a
couple of results or even an internal server error. Let's implement
those three cases as scenarios. Create a new file
`robohydra/search/index.js` with the following contents:

{% highlight javascript %}
var RoboHydraHead           = require("robohydra").heads.RoboHydraHead,
    RoboHydraHeadStatic     = require("robohydra").heads.RoboHydraHeadStatic,
    Response                = require("robohydra").Response;

exports.getBodyParts = function(conf) {
    return {
        heads: [
            new RoboHydraHeadStatic({
                path: '/foo',
                content: "This is the default behaviour of /foo"
            }),
            new RoboHydraHeadStatic({
                path: '/bar',
                content: "This is always available, regardless of the current scenario"
            })
        ],
        scenarios: {
            noResults: {
                heads: [
                    new RoboHydraHeadStatic({
                        path: '/foo',
                        content: {
                            "success": true,
                            "results": []
                        }
                    })
                ]
            },

            twoResults: {
                heads: [
                    new RoboHydraHeadStatic({
                        path: '/foo',
                        content: {
                            "success": true,
                            "results": [
                                {"url": "http://robohydra.org",
                                 "title": "RoboHydra testing tool"},
                                {"url": "http://en.wikipedia.org/wiki/Hydra",
                                 "title": "Hydra - Wikipedia"}
                            ]
                        }
                    })
                ]
            },

            serverProblems: {
                instructions: "Make a search with any non-empty search term. The client should show some error messaging stating the server couldn't fulfill the request or wasn't available",
                heads: [
                    new RoboHydraHead({
                        path: '/.*',
                        handler: function(req, res) {
                            res.statusCode = 500;
                            res.send("500 - (Synthetic) Internal Server Error");
                        }
                    })
                ]
            }
        }
    };
};
{% endhighlight %}

Once saved, run RoboHydra like so:

    robohydra -n -P search

You can see the available scenarios, which one is active (if any) and
start and stop them in the [scenario admin
interface](http://localhost:3000/robohydra-admin/scenarios). When
starting RoboHydra there won't be any active scenario, so
[`/foo`](http://localhost:3000/foo) will say "This is the default
behaviour of /foo". However, if we go to the scenario interface and
start any of the scenarios, we'll have the desired behaviour in
`/foo`. Go now to the scenario interface and experiment a bit with the
results you get when starting the different scenarios.

Note how when you start the last scenario, RoboHydra will show some
instructions and some expected behaviour (the `instructions` field is
interpreted as Markdown). This can be very handy if you want to build
some kind of acceptance test suite of your client.  And in case you
want to automate the testing of the client, you will also want to
automate the starting/switching of scenarios in RoboHydra. In that
case, you can use the RoboHydra [REST API](../../rest/).


Assertions
----------

If you're preparing a more formal test suite for your project, you may
want to not only check how the client behaves with different responses
from the server, but also if the client requests are well-formed and
correct. RoboHydra heads can contain assertions that will be counted as
part of the current active scenario.

Let's say we are still testing the client of that search engine in the
previous section, and we want to make sure we don't have character
encoding problems. Thus, we'll add a scenario that checks that the
client sent a correctly formed, UTF-8 search string. We can start by
adding a new scenario, `nonAsciiSearchTerm`, with the following
definition:

{% highlight javascript %}
exports.getBodyParts = function(conf, modules) {
    var assert = modules.assert;

    return {
        heads: [
            // ...
        ],
        scenarios: {
            // ...

            nonAsciiSearchTerm: {
                instructions: "Search for the string 'blåbærsyltetøy'.\n\nYou should _get one search result_ and it should be _displayed correctly_.",
                heads: [
                    new RoboHydraHead({
                        path: '/foo',
                        handler: function(req, res) {
                            res.headers['content-type'] =
                                'application/json; charset=utf-8';

                            // Only for RoboHydra >= 0.3
                            if (assert.equal(
                                req.queryParams.q,
                                "blåbærsyltetøy",
                                "Character encoding should be ok")
                               ) {
                                   res.send(JSON.stringify({
                                       success: true,
                                       results: [
                                           {"url":   "http://example.com",
                                            "title": "Blåbærsyltetøy'r us"}
                                       ]}));
                            } else {
                                res.send(JSON.stringify({
                                    success: false,
                                    results: []
                                }));
                            }
                        }
                    })
                ]
            }
        }
    }
}
{% endhighlight %}

Note that now the `getBodyParts` function accepts a second parameter,
`modules`. This second parameter is an object with available modules
as its properties. The only module available as of today is `assert`:
an object with all the functions in the standard, [`assert`
module](http://nodejs.org/docs/latest/api/assert.html) from Node. This
version, though, is special in two ways: first, it's tied to the
server, which allows RoboHydra to fetch the results; second, test
failures won't throw an exception, but instead return false. This
behaviour is much more useful because it allows you to respond with
whatever content you want in case of failure.

Now, if you start the `nonAsciiSearchTerm` scenario and make a request
with the wrong string
(eg. [http://localhost:3000/foo?q=blaabaersyltetoy](http://localhost:3000/foo?q=blaabaersyltetoy)),
you'll get an error response, and see the test failure in the
[scenario index
page](http://localhost:3000/robohydra-admin/scenarios). If you send
the correct string
(eg. [http://localhost:3000/foo?q=blåbærsyltetøy](http://localhost:3000/foo?q=blåbærsyltetøy)),
however, you'll see the one-result response and the test pass in the
scenario index page. In case you want to access this information in an
automated fashion, you can get the results in JSON format at the URL
[http://localhost:3000/robohydra-admin/rest/test-results](http://localhost:3000/robohydra-admin/rest/test-results).


Multi-user RoboHydra
--------------------

Especially when you are using RoboHydra for test suites inside a
company, you might want to have a single RoboHydra installation for
the whole organisation. That is, instead of every developer or tester
having one in their own private workstation, having one shared
RoboHydra server for everyone.

If you used RoboHydra as we have seen so far, that would be a problem
because when two users tried to use the test suite at the same time,
the second user would reset the active scenario while the first user
is still testing someting. To solve that problem, the RoboHydra server
permits having more than one "hydra" ("hydra" being a collection of
heads with their own state) and choose between them using the
*summoner*. The idea is that upon receiving a request, the summoner
will give a different hydra to every user (eg. by checking cookies):
that way many users can use the same RoboHydra server without
interfering with one another. The default *summoner* will always
summon the same hydra (ie. it's for a single user), but you can
provide the function that receives the incoming request object and
returns the name of the hydra that should process that request.

Let's make a simple, multi-user RoboHydra server. We'll start by
deciding how we're going to tell the different users apart: although a
common way to do so in the real world would be to use cookies, as a
simplification we'll just use the browser user agents. Create a file
`robohydra/ua-summoner/index.js` with the following contents:

{% highlight javascript %}
exports.getBodyParts = function() {
    return {};
};

exports.getSummonerTraits = function() {
    return {
        hydraPicker: function(req) {
            var ua = req.headers['user-agent'] || '<empty>';
            // Avoid very long hydra names, they look ugly on the admin UI
            return ua.length > 50 ? ua.slice(0, 50) + '…' : ua;
        }
    };
};
{% endhighlight %}

In this case every user agent will be considered a different user. In
other words, every browser you have installed (including different
versions!) will get its requests served by a different hydra. To test
this, start RoboHydra like `robohydra -P ua-summoner`, fire two or
three different browsers and go to
[/robohydra-admin/](http://localhost:3000/robohydra-admin/) with all
of them. You'll notice on the bottom left that the hydra name is
different for every browser. If you create new heads with one browser
using the admin UI, the other browser will _not_ see those new
heads. That means all browsers you try with can work independently of
each other, but all of them using the same server!


Conclusion
----------

And this is the end of the advanced RoboHydra tutorial. Now you have
learned about all features and it's just a matter of experimenting and
creating your own plugins.

For more information, check the [reference documentation](../..).
