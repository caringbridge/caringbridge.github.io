---
layout: post
title:  "How We Roll"
date:   2014-06-12 23:13:00
categories: practice
---

Here at CaringBridge, our website gets about 1 million unique visitors per day.
That's a lot of traffic for a nonprofit organization our size, and we're
frequently asked how we make it all work. We'll be using this blog to talk
about the challenges we've faced over the years and what we learned from
overcoming them. Let's get things started with an overview of our technology
stack.

## Backend ##

At the core of everything sits a web application that we just call `platform`.
It's written almost entirely in [PHP][php] and leans heavily
[Zend Framework 2][zf2] for all the plumbing we don't want to write ourselves.
Along with the code that powers all CaringBridge websites, `platform`
contains:

   * Our framework for managing Users and Profiles.
   * A web service API for our iPhone and Android apps.
   * A custom-built CMS that allows us to tailor content to users who meet
     specific criteria (e.g. "Show this help message to users who are authors
     of a site but haven't posted any journal entries yet.").
   * A custom-built A-B testing framework, which allows us to guage the
     effectiveness of new content and features. ([A-B testing is awesome][patio11]
     and you should be doing it.)
   * A job queue framework for tasks that are too time-intensive to happen
     in a web request.

The `platform` applications runs on [Apache][apache] and [Zend Server][zs].
We're aware that all the cool kids use [nginx][nginx], but Apache works just
fine for us.

CaringBridge would simply cease to function without an effective caching
mechanism. We use [APC][apc] to cache application code and [Twemcache][twemcache]
to cache data. We also use Twemcache as our session store.

Our job queue framework is built around [RabbitMQ][rabbitmq]. We used to use
[Gearman][gearman] but encountered bizarre problems when it was under heavy load.
We've had nothing but good experiences with RabbitMQ.

## Databases ##

For several years, we used [MySQL][mysql], as our main database. As our traffic
increased over time, we had to denormalize more and more of our table structure
to avoid performance bottlenecks. Eventually, we took the plunge and migrated
everything over to [MongoDB][mongo], a document-oriented NoSQL database.

The main difference between Mongo and a traditional RDBMS like MySQL is that,
in Mongo, there's no such thing as a join. At all. Instead of joining two
tables together with a foreign key, Mongo encourages you to nest related
records together into a single document.

For instance, a blog post might look like this:

{% highlight js %}
{
  "_id": ObjectId("521ccef0281f9bac1d000001"),
  "title": "How We Roll",
  "date": ISODate("2014-06-12T23:13:57.315Z"),
  "body": "This is the post content",
  "tags": ["php", "mongo", "rabbitmq", "caringbridge"],
  "comments": [
    {
      "date": ISODate("2014-06-12T23:13:57.315Z"),
      "body": "First post!",
      "author": "Stewi",
      "email": "stewi@caringbridge.org",
    },
    {
      "date": ISODate("2014-06-12T23:13:57.315Z"),
      "body": "Check out Stewi's movie http://youtu.be/BgFyAasYA8Y",
      "author": "Aaron",
      "email": "aaron@caringbridge.org",
    },
  ]
}
{% endhighlight %}

Instead of having separate `tag` and `comment` tables, as you might have in
MySQL, the tags and comments are stored in the same record as the blog post
itself. This way, you can avoid expensive joins and fetch everything you need
to display a blog post with a single, simple query.

{% highlight js %}
db.post.findOne({_id: ObjectId("521ccef0281f9bac1d000001")});
{% endhighlight %}

Our friend and former coworker [Andrew Kandels][papa] created an entity
framework called [Contain][contain] and wrote a [Data Mapper][dm] that
allows us persist Contain entitie as Mongo documents.

## Frontend ##

We believe in [progressive enhancement][ala] and [responsive web design][rwd],
and because we're mission-focused instead of profit-focused, we're able to
support old browsers, broken browsers, mobile browsers, screen readers, and
other browsing software that slips through the cracks at most for-profit
companies.

We jumped on the [Twitter Bootstrap][twbs] bandwagon almost soon as it was
released. Bootstrap critics complain that all&mdash;or at
least most&mdash;Bootstrap sites look the same, but it doesn't have to be
that way. We've customized the Framework with our own colors, typography, and
design language, but we like to be able to fall back on reasonable, well-thought-out
defaults when we can.

We rely heavily on [SASS][sass] and [Compass][compass] to build our stylesheets,
and use Thomas McDonald's [`bootstrap-sass`][bs-sass] Ruby gem to customize
Bootstrap's styles to meet our requirements. We were, at first, skeptical of CSS
preprocessors, but after more than a year of use, it's nearly impossible to
imagine how we ever got by without it. With SASS, you can define variables:

{% highlight scss %}
$serif-font: "Palatino Linotype", Palatino, Georgia, "Times New Roman", serif;
$sans-serif-font: "Helvetica Neue", Arial, Helvetica, sans-serif;

body {
  font-family: $serif-font;
}

h1,h2,h3 {
  font-family: $sans-serif-font;
  font-weight: bold;
}
{% endhighlight %}

And mixins:

{% highlight scss %}
@mixin border-radius($radius) {
  -webkit-border-radius: $radius;
  -moz-border-radius: $radius;
  -ms-border-radius: $radius;
  border-radius: $radius;
}

.box {
  @include border-radius(8px);
}
{% endhighlight %}

And that's just scratching the surface.

Because of our commitment to progressive enhancement, we've identified a core
subset of features that absolutely must work, even if a user's browser can't
execute JavaScript. That said, we're all about using JavaScript to enhance our
core experience. In the past, we had trouble finding a way to structure and
unit test our JavaScript, but now [RequireJS][rjs] and [Jasmine][jasmine] take
care of that for us.

We're still a bit skeptical of single-page apps, but we use
[Backbone][backbone]'s models in several places, where we have
[REST][rest]ful AJAX calls back to our server.

## Wow, This is Getting Long ##

That's a lot to take in, and it doesn't cover everything that's running under
the hood, but it's a good overview of what it takes to keep our service running.
In the future, we'll have some posts that talk about nuances of getting these
tools to work with each other and some discussion of the strategy and
architectural principles that guide our decision-making. Until then, Happy Hacking!

[php]: http://php.net/
[zf2]: http://framework.zend.com/
[patio11]: http://www.kalzumeus.com/category/ab-testing/
[apache]: http://httpd.apache.org/
[zs]: http://www.zend.com/en/products/server/
[nginx]: http://nginx.org/
[apc]: http://www.php.net//manual/en/book.apc.php
[twemcache]: https://blog.twitter.com/2012/caching-with-twemcache
[rabbitmq]: http://www.rabbitmq.com/
[gearman]: http://gearman.org/
[mysql]: http://www.mysql.com/
[mongo]: http://www.mongodb.org/
[papa]: http://andrewkandels.com/
[contain]: https://github.com/andrew-kandels/contain
[dm]: http://martinfowler.com/eaaCatalog/dataMapper.html
[twbs]: http://getbootstrap.com/
[ala]: http://alistapart.com/article/understandingprogressiveenhancement/
[rwd]: http://alistapart.com/article/responsive-web-design/
[sass]: http://sass-lang.com/
[compass]: http://compass-style.org/
[bs-sass]: https://github.com/twbs/bootstrap-sass
[rjs]: http://requirejs.org/
[jasmine]: http://jasmine.github.io/
[backbone]: http://jasmine.github.io/
[rest]: http://en.wikipedia.org/wiki/Representational_state_transfer
