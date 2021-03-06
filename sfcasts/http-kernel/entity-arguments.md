# How Entity Controller Arguments Work

In `ArticleController::show()` we're using the `$slug` argument to manually query
for the `Article` object and triggering a 404 page if needed. But
*originally* this code looked different: instead of a `$slug`
argument, we had an `$article` argument type-hinted with the `Article` entity
class. We didn't need to make the query because something else was doing it for
us. The same is true for the 404 logic.

I love this feature! But the question for *us* is: how does this work? What is
*allowing* us to have this `$article` argument. *Something* is *noticing* that
we're type-hinting an argument with an entity class and is automatically querying
for it based on the `{slug}` wildcard. But where is that code?

At first, you might think this is another argument value resolver. But, there's
nothing in that list that mentions "doctrine" or "entity". In reality, this is
working via a *different* system.

If we refresh now, the page still works. But if you change the URL to a
slug that *won't* be found, the error gives us a hint:

> `App\Entity\Article` object not found by the `@ParamConverter` annotation

## ParamConverterListener

This error is coming from a class called `DoctrineParamConverter`, which if
you look further down the stack trace, is called by some `ParamConverterListener`.

Let's start by checking out that class. I'll hit Shift + Shift to open
`ParamConverterListener.php`. Ah - this class comes from
`SensioFrameworkExtraBundle`. The first thing I want you to notice is that this
implements `EventSubscriberInterface`. Yep! This is an *event* listener and it
listens to the `kernel.controller` event: the event that's dispatched *after*
the controller is determined, but *before* the controller is *called*. And also
before the arguments are determined.

That makes sense! If this class is going to do some magic on the arguments
to our controller, it's going to need to know which controller is about to be
called so it can *look* at its arguments.

We're not going to study the details of this class *too* closely, but we can
the basic idea pretty easily. This function loops over all of the parameters - all
of the arguments of the controller. That `$param` is a `ReflectionParameter`
which holds info about that argument. Most importantly, it knows what *class*
the argument is type-hinted with.

## DoctrineParamConverter

Anyways, that method collects info about each argument and then, eventually -
if you look at the stacktrace - it executes something called
`DoctrineParamConverter`. That's where the magic happens. I'll hit Shift + Shift
to open that file: `DoctrineParamConverter.php`.

Hmm. This starts by getting the `$class`: that's the type-hint on the argument.
Then, it tries different ways of querying for that. For example, `$this->find()`
tries to use the `id`: it tries to see if the primary key is a wildcard in the URL,
gets the entity manager for that class and ultimately tries to call a method to
make that query.

This feature has a lot of options and can query for your entity in a lot of
different ways. But we're not going to get into the weeds about all of that now.

The *really* cool part is that - no matter *how* it finds your entity - if we
follow the logic to the bottom of `apply()`, *eventually* it takes that `Article`
object and sets it onto the `$request->attributes`! The `$name` variable is the
*name* of the argument - `article` for us - and `$object` will be the full
entity that was fetched from the database.

SensioFrameworkExtraBundle is *full* of magic like this and *all* that magic works
via *listeners*. If you want to know more about how one of its features works,
find the listener that it's using. Oh, and if you're wondering why this
`DoctrineParamConverter` *doesn't* just use the "argument value resolver" system,
the answer is that it *pre-dates* it. It may, some day, be converted to use it.

Next, let's start talking about a *fascinating* topic that we've already seen
a few times. I want to talk about sub requests.
