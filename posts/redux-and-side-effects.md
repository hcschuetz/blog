Redux and Side Effects
======================

I really appreciate the Redux approach of avoiding side effects in
application code.  Of course in the end somewhere deep inside Redux
there must be a side effect replacing the old state of the store with a
new one.  But that's shielded very well from the application programmer.

Given the fact that the Redux way provides the advantages of declarative
functional programming, the invocation of side effects in Redux
middleware feels like a hack.  The smell is not about monkeypatching.
After all, Redux provides an API for plugging in middleware and the
Redux tutorial explains how to use it for side effects.  Often a new way
of programming starts out as a hack.  Later the hack is standardized and
can then be considered a pattern.  But still it doesn't feel right for
me here.  The smell is about the imperative action handlers in the
middleware, which can make it hard to reason about program behavior.


An Example Use Case
-------------------

So what would I suggest?  I do not yet have a complete solution, but I
have an approach to follow, which I'd like to explain with an example.
Assume you have a connection to some database.  In our application we
have a paginated view on some list of data and the database can be
queried for the list items in a given range.

With Redux middleware the typical solution would work like this:

- The UI emits an action saying "Fetch items 60 to 69 and when you have
  them, emit another action to store them in the state".
- When the (thunk-handling or promise-handling) middleware receives the
  action, it launches the query.  A handler function for the result is
  attached to the database connection.
- An auxiliary action might be propagated to note in the state that the
  query is ongoing.
- When the query returns with some result, the handler function is
  invoked.  It will emit an action to the store which will add items 60
  to 69 to the state.

(Even though I say that something is noted in or added to the state I of
course mean that this is computed functionally by the reducer.)

Now let's make the problem a bit harder and impose that the database
connection is single-threaded.  That is, you cannot send a query or
update operation before the previous operation has completed.  The
action now has to check if there is some ongoing database operation.  If
no, things proceed as above.  Otherwise the action is appended to some
queue.  Whenever a database operation completes, there is (in addition
to the steps described above) a check whether the queue is empty.  If
not, the top element of the queue is taken and executed as if it had
just arrived.

This is already a bit more difficult but still manageable.  In
particular we could encapsulate the queuing in the database API to
relieve the application programmer from the extra burden of queue
management.

Next, let's assume that the user opens the list and already knows that
the interesting item is a few pages down the list.  So she/he presses
the "Page Down" button a few times.  With our naive implementation we
would have to execute database queries for all the pages down to the
interesting one, which may take quite a while.  It would be better to
notice that queries for earlier pages have become irrelevant once the
user pressed the "Page Down" button another time and to remove these
queries from the queue before they would be sent to the database.  Now
things become a bit hairy.

I'm not going into the details of how this could be solved with a
thunk/promise-middleware approach, but you can imagine that it becomes
quite cumbersome to correctly determine the queue elements that may be
deleted.  I'm rather going to describe my suggested approach.


Suggested Solution
------------------

I think we need no middleware here.  The state already contains
information which database items the UI would like to display.  In our
example this is at which page of the paginated list we are.

There is some standard "database manager" code which invokes an
application-provided "query function"

- whenever the state changes and the database connection is idle
- and also whenever the database connection has completed an operation.

The query function

- scans the state for any data that the UI would like to display but
  that is not available
- decides (by whatever criteria) which missing data is most urgent to
  fetch (In the example, this would be the item range corresponding to
  the selected page minus the items that are already available.)
- returns the respective query together with a callback to be invoked on
  the query results (or null if there is nothing to fetch).

The database manager sends the query to the database.  (Notice that the
database connection is idle at this point.)

When the query returns a result, the database manager invokes the
callback.  That callback will typically emit an action telling the
reducer to add the retrieved data to the state.

In our example this would lead to the following behavior:

- When the user opens the list the first page is marked as
  "interesting".
- Provided that the database connection is idle, the database manager
  will get a query for the first page from the query function and send
  it off to the database.
- While the database is busy, the user presses "Page Down" several
  times.  This causes the state to declare another page to be
  "interesting".
- When the query for the first page returns the callback will emit an
  action causing the reducer to add the returned data to the state.  The
  reducer might either notice that this data is no more interesting and
  discard it, or it might store the data nevertheless and leave it to a
  "cleanup" action to remove it.
- The database manager will invoke the query function again and get a
  query for the page currently selected by the user.

This way it was straight-forward to skip queries for all the
intermediate pages, which otherwise might have slowed down the process
considerably.

Further notes:

- The query function might even optimize database accesses by combining
  data requests from different parts of the UI state into a single
  query.
- If there's nothing really urgent to fetch, the query function might
  choose to do some pre-fetching or to re-fetch potentially stale data.
- There may also be a standard "eviction service" which regularly emits
  a "cleanup" action telling the reducer to remove data that's no more
  needed.  Alternatively, the "normal" reducer behavior may already
  ensure that the state never contains superfluous data.
- The approach should also work for backends with different
  characteristics.
- I have treated the retrieval of data, but the approach should also
  work for storing data.
- Database errors can be handled by the callback returned by the query
  function or by a second callback.
- Of course the database manager can be adapted to other backend
  technologies.
- No extension to Redux is needed.  The database manager simply
  subscribes to the store.

But most importantly: **The user-provided query function is fully
functional/declarative**.  It just maps the current state to a query.  The
actual side effects are left to the standard database-manager utility.

Well, there's still the callbacks for the query results, which will emit
actions.  But this is completely analogous to DOM events emitting
actions in a React-based UI.

There's actually quite some analogy between the database manager and
React.  It would be interesting to investigate how far we can get this
analogy (or how far it makes sense).  Some questions:

- What's the counterpart to VDOM diffing?
- Shouldn't a React UI just be a plain subscriber to the Redux store?
  (Maybe it internally already is.)
- ...


Next Steps
----------

Open questions include:

- For which databases (or other backends) should we have managers?
  (IndexedDB and GraphQL come to mind.)
- How much code and concepts can be shared between different backend
  managers?

I'm afraid that I will not have the time to implement a "standard"
database manager anytime soon.  But I will follow these ideas during
application development and see how far they get me.

Perhaps someone feels encouraged to follow this approach and maybe even
to provide a library supporting it.
