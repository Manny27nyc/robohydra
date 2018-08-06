---
layout: default
---

RoboHydra server tutorial
=========================

RoboHydra is an HTTP server designed to help you develop and test clients
of client-server applications. You can start an empty RoboHydra server
by typing (the `-n` tells RoboHydra to not read any configuration file):

    robohydra -n

Because it's empty it will return 404 for every request, except for
the admin interface, which is always available at
[`/robohydra-admin`](http://localhost:3000/robohydra-admin). You will
find this admin interface handy to understand the current behaviour of
your RoboHydra server.


Your first canned response
--------------------------

Now, let's assume you want the path `/foo` to always return the
following JSON data:

{% highlight json %}
{"success": true,
 "results": [{"url": "http://robohydra.org",
              "title": "RoboHydra testing tool"},
             {"url": "http://en.wikipedia.org/wiki/Hydra",
              "title": "Hydra - Wikipedia"}]
{% endhighlight %}

You can do that very easily: simply go to the [admin
interface](http://localhost:3000/robohydra-admin), find the "Create a
new head" section at the bottom and type the path and the content in
the appropriate fields, and click on "Create". Don't worry about the
"Content-Type" field: by default, if the content can be parsed as
JSON, it will be `application/json`.

<figure>
  <img src="robohydra-admin1.png" />
  <figcaption>Figure 1: creating a dynamic head from the RoboHydra
  admin interface</figcaption>
</figure>

If you now visit [your new head](http://localhost:3000/foo), you
should receive the above data in the response, and with the correct
Content-Type. All other paths will still yield 404. However, if you
kill this RoboHydra and start a new one, that head will be lost. Don't
worry though, you can easily write your heads in files (plugins) so
you can load them whenever you need them.


Your first plugin
-----------------

So far we have seen how to create "heads" dynamically in our RoboHydra
server, by using the admin interface. A "head" is an object that
monitors a given URL path pattern and defines how to process requests
for those URLs (eg. always return a given response for the URL path
`/orders/.*`, or alternatively return 404 and 200 for the URL
`/rest/widgets/3`).

As there are many heads that you want to keep for later and use over
and over, RoboHydra allows you to write "plugins", which are files
describing one or more heads.

For example, if you wanted to keep the above head for later, you could
write a simple plugin like this:

{% highlight javascript %}
var RoboHydraHeadStatic = require("robohydra").heads.RoboHydraHeadStatic;

exports.getBodyParts = function(conf) {
    return {
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
    };
};
{% endhighlight %}

In this case, our first plugin has a single head of type
`RoboHydraHeadStatic`. You can use the `RoboHydraHeadStatic` class
when you only want to return a certain static response regardless of
the incoming request. Now, save the above text in a file
`robohydra/firstplugin/index.js` and start RoboHydra like so:

    robohydra -n -P firstplugin

A head listening in `/foo` will be available from the start. Another
possibility would be to create a configuration file (say,
`first.conf`) with the following contents:

{% highlight json %}
{"plugins": ["firstplugin"]}
{% endhighlight %}

And then start RoboHydra as `robohydra first.conf`. The end result is
the same. With a configuration file, however, is easier to change the
list of plugins to be loaded, and you can configure other things like
SSL certificates (in the case you want RoboHydra to server HTTPS
requests). See the [configuration file
documentation](../configuration/) for more
information.


More flexibility in request handling
------------------------------------

As sending only static, fixed responses is no fun, you can also write
your own Javascript functions to process the requests and build the
responses. For example, say you want to wait for one second before you
send the response back to the client. To do that, modify the `require`
line and `heads` section in your plugin to add a second head like so:

{% highlight javascript %}
var RoboHydraHeadStatic = require("robohydra").heads.RoboHydraHeadStatic,
    RoboHydraHead       = require("robohydra").heads.RoboHydraHead;

exports.getBodyParts = function(conf) {
    return {
        heads: [
            new RoboHydraHeadStatic({
                // ... Same as before ...
            }),

            new RoboHydraHead({
                path: '/slow',
                handler: function(req, res) {
                    setTimeout(function() {
                        res.send("Some slow response");
                    }, 1000);
                }
            })
        ]
    };
};
{% endhighlight %}

If you now make a request to http://localhost:3000/slow, you will get
the text response `Some slow response` after one second.


Accessing the request data
--------------------------

Now let's say that we want to configure how slow the request is going
to return, by specifying URLs like `/slow/1000`, `/slow/5000` and so
on. In this case, RoboHydra allows you to specify URL paths like
`/slow/:millis` which will match any URL path fragment after
`/slow`. The following handler function will allow you to configure
the wait in this way:

{% highlight javascript %}
new RoboHydraHead({
    path: '/slow/:millis',
    handler: function(req, res) {
        setTimeout(function() {
            res.send("Some slow response");
        }, req.params.millis);
    }
})
{% endhighlight %}

But what about URLs like `/slow/?amount=3000`? In that case, you have
the GET parameters avaiable as properties of the object
`req.queryParams`. Similarly, the POST information and the raw body of
the request are available as `req.body` and `req.rawBody`
respectively. The former is an object for
`application/x-www-form-urlencoded` and `application/json` requests,
plain text for `text/plain` and `text/html`, and null otherwise. The
latter is a [`Buffer`](http://nodejs.org/docs/latest/api/buffer.html)
object. This head would match the GET-parameter-style URLs:

{% highlight javascript %}
new RoboHydraHead({
    path: '/slow',
    handler: function(req, res) {
        setTimeout(function() {
            res.send("Some slow response");
        }, req.queryParams.millis || 1000);
    }
})
{% endhighlight %}


Other handy kinds of heads
--------------------------

Apart from `RoboHydraHeadStatic` and the generic `RoboHydraHead`,
there are other, more specific head classes. In this section we will
see two of them in action: `RoboHydraHeadFilesystem` and
`RoboHydraHeadProxy`. As you can guess, the former serves static files
from the filesystem, while the latter proxies requests to another URL.

One of the many ways in which you can combine these two heads is
having a RoboHydra that proxies everything to another server, except
certain paths that contain your own, local copies of some files. This
can be useful when you have frontend developers working on some site:
if you don't want to make them have their own local installation of
the backend, they can simply have a copy of the CSS and Javascript
files they maintain, and use RoboHydra to serve their local files for
requests to, say, `/css` and `/js` while it proxies all the rest to
the common, development server.

Let's demonstrate this with a simple example. Let's say you are the
new designer for the RoboHydra homepage, and want to try your new
assets. You can make images locally, and see how they would look in
the production RoboHydra homepage by using a simple reverse
proxy. Download the [new example image](usecases-teaser.v1.jpg) for
the comic link and the [robohydra logo](/static/img/robohydra.png) and
save them in a directory `new-assets`. Now, write this simple plugin:

{% highlight javascript %}
var RoboHydraHeadFilesystem = require("robohydra").heads.RoboHydraHeadFilesystem,
    RoboHydraHeadProxy      = require("robohydra").heads.RoboHydraHeadProxy;

exports.getBodyParts = function(conf) {
    return {
        heads: [
            new RoboHydraHeadFilesystem({
                mountPath: '/static/img',
                documentRoot: 'new-assets'
            }),

            new RoboHydraHeadProxy({
                mountPath: '/',
                proxyTo: 'http://robohydra.org'
            })
        ]
    };
};
{% endhighlight %}

Note that the first head that matches the request dispatches it, so
the order is important! Now save the plugin as
`robohydra/rhimages/index.js` and start RoboHydra as `robohydra -n -P
rhimages`. You should see the RoboHydra front page as normal, except
for the new image on the top-right.

Now you know all the basic functionality RoboHydra offers. If you want
more information, you can read and follow the <a
href="advanced/">advanced tutorial</a>.
