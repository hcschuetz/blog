Three Notes on Redux
====================

Declarative Backend Operations
------------------------------

See [separate blog post](../posts/redux-and-side-effects.md).


Nested Actions for Nested States
--------------------------------

> ```
> {type: WITH_ITEM, index: i, action: {... some "subaction" to be applied to item i ...}}
> ```

> Problem: clash with middleware.  A wrapped action for a state
> component would not be recognized and intercepted by generic middleware.

> Other complex actions could invoke some async method (or create a
> promise) and provide sub-actions to invoke upon success and failure.

> Basic idea: Do not only provide primitive actions, but complex ones
> composable from other actions.  (This becomes a programming language.
> Why another language beyond JS?)


Object-Oriented State Structure
-------------------------------

The structure of the state is not described explicitly anywhere.  It's
just transformed by the reducers and read by React components.  The
reason for this is that it is represented as plain untyped JSON data.
So functionality concerning the state goes to reducers and React
components, which in turn may have to deal with different parts of the
state.  This easily leads to spaghetti code.

Consider a frequent case in business-software development: We need a new
field and code dealing with this field.  (This is much more frequent
than "architectural" changes, at least after the initial development
phase.)  It would be nice if we did not have to change code in lots of
different places.  Ideally all the relevant changes would happen in a
single place.  (Yes, there is a trade-off with other considerations, but
it looks like my consideration has not yet been taken into account in
the published "best practices" for redux such as the tutorial.)

Actually with redux one can use instances of specific classes to
represent the state.  Maybe one should.

This also allows to put core transformation functionality into methods
of these classes.  (Notice that these methods must still be purely
functional in that they do not modify "this" but rather return a
transformed version of "this".)  This would bring back the advantages of
encapsulation and object-oriented programming and can be used to avoid
spaghetti code.

We might use a convention that state classes provide (in addition to any
class-specific action) a method "reduce(action)".  Some standard
utilities might make use of this convention.

State classes might also provide utilities for the React components,
including relevant action creators.
