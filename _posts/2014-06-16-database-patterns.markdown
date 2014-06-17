---
layout: post
title: ActiveRecord, Data Mapper, and You
date: 2014-06-16
categories: theory
author: Stephen Van Dahm
---

When designing a Web application, one of the most important architectural
decisions you need to make concerns database abstraction. Slinging a bunch
of arbitrary database queries around leads to a yucky, untestable, and
unmaintainable mess, so most projects these days have a layer that translates
between records in your datastore and objects used in your app.

Martin Fowler, in [Patterns of Enterprise Architecture][paa], describes two
main design patterns that address this need:

   * [ActiveRecord][ar], in which "an object that wraps a row in a database
     table or view, encapsulates the database access, and adds domain logic
     on that data."
   * [Data Mapper][dm], which calls for "a layer of Mappers that moves data
     between objects and a database while keeping them independent of each
     other and the mapper itself."

At CaringBridge, we've tried both of these approaches both with traditional
and document-oriented databases, so we've seen the advantages and
disadvantages of each approach.

## ActiveRecord ##

ActiveRecord is the simplest pattern to use, and we used it at CaringBridge
for several years. In ActiveRecord, a model class roughly corresponds to an
SQL table, and an instance of that class roughly corresponds to a row within
that table.  Class methods are usually responsible for loading records from
the database and mapping them into objects in memory. The objects themselves
are responsible for saving modifications back to the database and handling any
business logic tied to the data represented in the object.

[The ActiveRecord implementation built into Ruby on Rails][arror] is probably
the most famous and influential, and it's been copied by countless other
frameworks designed for numerous languages. In Rails, you'd select a record like
this:

{% highlight ruby %}
user = User.find(12345)
{% endhighlight %}

To construct the `user` object, Rails would probably run a query similar to
this one:

{% highlight sql %}
SELECT *
FROM `users`
WHERE `id`=12345
LIMIT 1;
{% endhighlight %}

And it's easy enough to change a record and save it back to the database:

{% highlight ruby %}
user.nickname = 'Stewi'
user.save # Note this method. We're not done with it yet.
{% endhighlight %}

Behind the scenes, Rails probably generates a query similar to this one:

{% highlight sql %}
UPDATE `users`
SET `nickname` = 'Stewi'
WHERE `id` = 12345;
{% endhighlight %}

And that's awesome! Not only do you avoid slinging database queries around
all over your app, the SQL generator can protect you against [SQL injection
attacks][xkcd].

But it's not all wine and roses. In practice, ActiveRecord has caused us a lot
grief. Sometimes it might not even be feasible to represent a kind of entity
with a single database table, and even when it is, tying your database structure
closely to the classes and objects in your application can make it difficult to
modify that structure later on.  Futhermore, there's a lot to be said for
separating your data retrieval and storage logic from the other business logic
of the app.

## Data Mapper and the Single Responsibility Principle ##

The Data Mapper pattern is more complex than ActiveRecord but addresses many
of its shortcomings. For each entity in your business domain, you just define
an entity class with simple properties and business logic that operates on those
properties. Since we've been thinking in Ruby, we might start with something
like this:

{% highlight ruby %}
class User
  attr_accessor :first_name
  attr_accessor :last_name
  attr_accessor :nickname
  attr_accessor :email

  def full_name
    return "#{first_name} #{last_name}" unless nickname
    return "#{first_name} '#{nickname}' #{last_name}"
  end
end
{% endhighlight %}

And that's it! User objects don't know where they come from, how they get saved
to the database, or even *if* they get saved to a database. It's just data and
logic.

You'd then need a mapper that can translate these objects to database records
and back again. Maybe you'd have something like:

{% highlight ruby %}
user = @user_mapper.find_by_email('stewi@example.com')
user.nickname = 'Stewi'
@user_mapper.persist(user)
{% endhighlight %}

In the simple example above, it's hard to see how this is any kind of improvement,
but in a complex web application, the [Single Responsibility Principle][srp] is
a lifesaver. Separating data entities from the means by which those entities
are persisted gives you an enormous degree of architectural flexibility, which
is what allows your application to grow and change in response to technical
and business needs.

The one downside I've seen to using Data Mapper is that, since the mappers
can only retrieve and save records, they can't make any assumptions about how that
data is going to be used.  Assume that you're working on a blogging app.
Your database will likely have a `posts` table and a `comments` table. When your
mapper instantiates a `Post` object, should it also retrieve the associated
`Comment` objects? Just to be thorough, maybe it should. Then again, if you
aren't actually going to need the comments, maybe you don't want to incur
the performance hit. With ActiveRecord, you'd have a method like this, that will
fetch the comments on the fly if they are needed:
{% highlight ruby %}
post.comments.each do |c|
  puts c.body
  puts c.signature
end
{% endhighlight %}
But that's not possible with Data Mapper, so now you have to be sure that both
the `Post` mapper and `Comment` mapper are available wherever they might be
needed.

Sadly, We haven't seen a good way around this problem yet. Mapping relational
data to objects and classes is hard, and it seems that fancy design patterns
can only get us so far.

Then again, we don't use a relational database at CaringBridge.

## Data Mapper is Awesome with Document Databases ##

Document-oriented databases like make the object-data mapping problem
considerably more straightforward. In MongoDB, our database of choice, a blog
post might look like this:

{% highlight js %}
{
  "_id": ObjectId("536d0e73837657c50bcfdf57"),
  "title": "This is a post title",
  "body": "This is a post body",
  "comments": [
    {
      "body": "First post!",
      "signature": "Stewi",
      "email": "stewi@example.com"
    },
    {
      "body": "Uh...Second post?",
      "signature": "Aaron",
      "email": "aaron@example.com"
    }
  ]
}
{% endhighlight %}

Since the comments are nested inside the Post document itself, we don't need
to worry about how to handle related records. It's easy to imagine how this
record maps to business domain entities.


## Conclusion ##

We love the Data Mapper design pattern in principle, but we recognize that
it isn't always practical to separate database logic from business logic,
especially if your table structure is highly relational and you're worried
about taking a performance hit from unnecessary joins. ActiveRecord, despite
its limitations, will certainly do the trick, especially if you're building
a relatively simple application.

CaringBridge, however, has outgrown ActiveRecord, and since adopting the Data
Mapper pattern, it's hard to imagine going back.

In a future post, we'll talk about the [Contain][contain] framework and the
[contain-mapper][cm] library, which we use to define our data entities and map
them to Mongo, memcached, or any other datastore we choose.


[paa]: http://martinfowler.com/books/eaa.html
[ar]: http://martinfowler.com/eaaCatalog/activeRecord.html
[dm]: http://martinfowler.com/eaaCatalog/dataMapper.html
[arror]: http://guides.rubyonrails.org/active_record_basics.html
[xkcd]: http://xkcd.com/327/
[srp]: http://en.wikipedia.org/wiki/Single_responsibility_principle
[contain]: https://github.com/andrew-kandels/contain
[cm]: https://github.com/andrew-kandels/contain-mapper
