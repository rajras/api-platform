# Custom Resource PUT Operation

Back to the API documentation! Let's pretend that we also need to be able to
*update* a `DailyStats`. Maybe, if you're an admin, sometimes you need to
double-check the data and update it. Specifically, let's make it possible to
change the `totalVisitors` field.

Ok, cool! Updating is done via the `put` operation. Over in `DailyStats`, under
`itemOperations`, add `put`:

[[[ code('eac6d85e00') ]]]

Then, because this is the first time we will be *denormalizing*, copy
`normalizationContext`, paste, rename it to `denormalizationContext`
and set its `groups` to `daily-stats:write`:

[[[ code('87e5a7a96b') ]]]

Take that `daily-stats:write` group and put it above `totalVisitors`:

[[[ code('432a3280e6') ]]]

Let's give it a try! No, it won't work yet but... it will kind of seem like
it's working. Refresh the docs. There's our PUT operation. Execute the collection
operation so we can get a valid ID. Perfect. I'll copy this `2020-09-03`. Down in
the `put` operation... actually, let me scroll back up so we can see that the
`totalVisitors` is currently 1,500.

Down on the `put` operation, hit "Try it out", paste the date string as the `id`
and set `totalVisitors` to 500. Hit "Execute" and... it works! Wait, it worked?

No, it didn't *really* work. The deserialization process *did* update the
`DailyStats` object, which is why we see the correct number in the response. But
it's not *actually* saving. If you re-tried the get collection operation... yep!
It's *still* 1,500.

## Creating the Data Persister

What we're missing is a data persister for `DailyStats`.

Cool! We know how to make those! In the `src/DataPersister/` directory, create
a new PHP class and call it `DailyStatsPersister`, or, `DailyStatsDataPersister`
if you want to be more consistent than I'm being. Make this implement `DataPersisterInterface`:

[[[ code('07b065b15c') ]]]

Then go to the "Code"->"Generate" menu - or `Command` + `N` on a Mac - and select
"Implement Methods" - to add the three methods that we need:

[[[ code('693c04dc84') ]]]

As usual `supports()` is pretty easy: if a `DailyStats` is being saved,
*we* want to handle it. So, return `$data instanceof DailyStats`:

[[[ code('15e75a7817') ]]]

Next, we don't actually need `remove()` because we haven't added the `delete`
operation. Oh, but first, I mis-typed `$data`:

[[[ code('07a5a8a224') ]]]

Anyways, down in `remove()`, let's throw a new `Exception`:

> not supported

[[[ code('4c357d2ed1') ]]]

For `persist()`, to *truly* make this work, we should open the `fake_stats.json`
file and change its contents. But... doing that would be pretty boring and simple.
So instead, let's fake it and log a message inside `persist()` with the *new*
`totalVisitors` value.

To do that, add `public function __construct()` and autowire `LoggerInterface $logger`:

[[[ code('9292ef51c1') ]]]

Hit `Alt`+`Enter` and go to initialize properties to create that property and
set it:

[[[ code('53e860e1fb') ]]]

Down in persist, we know that the `$data` argument will be a `DailyStats` object.
Add a bit of PHPDoc above this: I don't need the `@return`, but I *do* want to say
that `$data` will be a `DailyStats` object:

[[[ code('2112feb8dc') ]]]

Inside the method, say: `$this->logger->info()`, `sprintf()`:

> Update the visitors to "%d"

And pass `$data->totalVisitors` for the wildcard:

[[[ code('d9a58d36cd') ]]]

Let's see if it works! Move back over. I still have my documentation open, so
let's just hit "Execute" again. When it finishes... okay! No error and it still
says `totalVisitors` 500.

To prove that our data persister *was* actually called, go down to the web debug
toolbar, hover over the AJAX icon, and open the last request that was made. I'll
open in a new tab.

This takes me to the *profiler* for that request. Go down to logs and... perfect!

> Update the visitors to "500"

So adding the `put` operation was... pretty simple! And we could also use this to
support the `POST` operation if we wanted to allow *new* items to be created.

Next: for `CheeseListing`, we added a bunch of built-in filters to allow users to
search and filter the results:

[[[ code('13a08c26f7') ]]]

But what if the built-in filters aren't enough? What if we need to add some
*custom* filtering logic? Let's do that next by *first* creating a custom filter
for a Doctrine entity and later creating a custom filter for our `DailyStats`
resource.
