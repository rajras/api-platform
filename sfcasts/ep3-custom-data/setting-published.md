# Publishing a Listing

One of the things that we *can't* do yet is publish a `CheeseListing`. Boo!

Right now, when we create a `CheeseListing` through our API, it always gets an
`isPublished=false` value, which is the default:

[[[ code('615420b005') ]]]

There is no way to change this... because `isPublished` isn't exposed as a field in
our API.

In the imaginary UI of our site, there will be a giant "Publish" button that a user
can click. When a user clicks that, we're obviously going to need to change the
`isPublished` field from `false` to `true`. But publishing in our app is *more*
than just updating a field in the database. Let's pretend that we *also* need to
run some custom code when a listing is published... like maybe we need to send a
request to ElasticSearch to *index* the new listing... or we want to send some
notifications to users who are desperately waiting for our cheese.

## Publishing: Custom Endpoint

So... how should we design this in our API? You *might* think that we need a custom
endpoint or "operation" in ApiPlatform language. Something like
`POST /api/cheeses/{id}/publish`.

You *can* do this. And we'll talk about custom operations in part 4 of this series.
But this solution would *not* be RESTful. In REST, every URL represents the address
to a unique resource. So from a REST standpoint, POSTing to
`/api/cheeses/{id}/publish` makes it look like there is a "cheese publish" resource...
and that we're trying to create a new one.

Of course, rules are *meant* to be broken. And ultimately, you should just get your
job done however you need to. But in this tutorial, let's see if we *can* solve this
in a RESTful way. How? By making `$isPublished` changeable in the *same* way as
*any* other field: by making a PUT request with `isPublished: true` in the body.

That part will be pretty easy. But running code *only* when this value changes
from false to true? That will be a bit trickier.

## Testing the PUT to update isPublished

Let's start with a basic test where we update this field. Open
`tests/Functional/CheeseListingResourceTest`, find `testUpdateCheeseListing()`, copy
the method, paste, and rename it to `testPublishCheeseListing()`:

[[[ code('5cf7ac8bb3') ]]]

Ok! I don't need 2 users: I'll just create one... and log in is that
user so that we have access to the PUT request:

[[[ code('a7eea4decf') ]]]

Thanks to the last tutorial, we already have security rules to prevent anyone
from editing someone else's listing. Down here, for the JSON body, send `isPublished`
set to `true`:

[[[ code('e94cf5a4da') ]]]

And... the status code we expect is 200:

[[[ code('537c42800e') ]]]

So here's the flow: we create a `User` - via the Foundry library - and then
create a `CheeseListing`. Oh, but we don't want that `published()` method: that's
a method I made to create a *published* listing:

[[[ code('ab6a23b0ce') ]]]

We definitely want to work with an *unpublished* listing. Anyways, we set the user
as the owner of the new cheese listing, log in as that user, and then send a PUT
request to update the `isPublished` field.

To make things more interesting, at the bottom, let's assert that the
`CheeseListing` *is* in fact published after the request. Do that with
`$cheeseListing->refresh()` - I'll talk about that in a second - and then
`$this->assertTrue()` that `$cheeseListing->getIsPublished()`:

[[[ code('bf9d509734') ]]]

`$cheeseListing->refresh()` is another feature of Foundry. Man, that
library just keeps on giving! Whenever you create an object with Foundry, it passes
you back that object but *wrapped* inside a `Proxy`. Hold `Command` or `Ctrl`
and click `refresh()`. Yep! A tiny `Proxy` class from Foundry with several
useful methods on it, like `refresh()`!

Anyways, `refresh()` will update the entity in my test with the latest data, and
then we'll check to make sure `$isPublished` is true.

Testing time! I mean, time to make sure our test fails! Copy the test method
name, spin over to your terminal, and run:

```terminal
symfony php bin/phpunit --filter=testPublishCheeseListing
```

We're hoping for failure and... yes!

> Failed asserting that `false` is `true`

Because... the `$isPublished` field is simply *not* writable in our API yet.

## Making isPublished Writable in the API

Let's fix that! At the top of the `@ApiResource` annotation, as a reminder, we
have a `denormalizationContext` that sets the serialization groups to
`cheese:write`:

[[[ code('56b6106063') ]]]

So if we want a field to be *writeable* in our API, it needs that group.

Copy that, scroll down to `isPublished`, add `@Groups({})` and paste:

[[[ code('b6035b0042') ]]]

Now, as long as this has a setter - and... yep there *is* a `setIsPublished()`
method:

[[[ code('99eb2c0e44') ]]]

It will be writable in the API.

Let's see if it is! Go back to your terminal and run the test again:

```terminal-silent
symfony php bin/phpunit --filter=testPublishCheeseListing
```

And... got it! We can now publish a `CheeseListing`! But... this
was the easy part. The *real* question is: how can we run custom code
*only* when a `CheeseListing` is published? So, only when the `isPublished`
field changes from `false` to `true`? Let's find out how next.
