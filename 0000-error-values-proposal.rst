Errors as values
================

.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. ticket-url:: Leave blank. This will eventually be filled with the
                ticket URL which will track the progress of the
                implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

Represent errors, warnings and suggestions as proper algebraic
data types, split by subsystem, with existential wrappers to
give us a way to store them uniformly.

Motivation
----------

Up until this day, the errors that GHC emits are represented as mere
textual documents. They often involve fancy layout constructs but GHC's
code never manipulates errors, warnings or suggestions as values of some
algebraic data types that are later rendered in the user's terminal.

This means that developers of IDE-style tooling (e.g
`Haskell IDE Engine <https://github.com/haskell/haskell-ide-engine>`_) have
to parse the error messages and warnings to implement some of their
features. This is not ideal and is made worse by the possibility that some
error messages might change in their contents or formatting, from a GHC
version to another. It would be a lot simpler for those developers to
put their hands on good old Haskell values that describe the errors,
warnings and suggestions that GHC reports for a given Haskell module.
The IDE tools and GHC API programs in general would then be able to
inspect those values and extract any relevant information without having
to parse any text, by simply traversing ADTS (e.g collect suggestions
and offer a feature that applies them all with a simple command or
keystroke). The ``ghc`` library would also come with code for rendering
those errors, making it easy for GHC API consumers to reuse all of GHC's
error printing infrastructure.

**Note**: the textual rendering of the errrors, warnings and suggestions
should remain identical, this proposal really is about GHC's
representation of errors.

Proposed Change Specification
-----------------------------

The current representation of errors and warnings in GHC is based on the
following data types.::

    data ErrMsg = ErrMsg {
            errMsgSpan        :: SrcSpan,
            errMsgContext     :: PrintUnqualified,
            errMsgDoc         :: ErrDoc,
            errMsgShortString :: String,
            errMsgSeverity    :: Severity,
            errMsgReason      :: WarnReason
            }

    data ErrDoc = ErrDoc {
            errDocImportant     :: [MsgDoc],
            errDocContext       :: [MsgDoc],
            errDocSupplementary :: [MsgDoc]
            }

    type MsgDoc = SDoc

    -- for completeness: warnings are represented with the same type
    -- as errors.
    type WarnMsg = ErrMsg

where ``SDoc`` is a type from GHC's pretty printing infrastructure that
represents configurable textual documents.

GHC then maintains a bag of ``ErrMsg`` and a bag of ``WarnMsg`` as
compilation proceeds and reports them when appropriate.::

    data TcLclEnv = TcLclEnv
      { ...
      , tcl_errs :: TcRef Messages     -- Place to accumulate errors
      , ...
      }

    type Messages        = (WarningMessages, ErrorMessages)
    type WarningMessages = Bag WarnMsg
    type ErrorMessages   = Bag ErrMsg

We propose to replace ``ErrDoc`` with several algebraic data types, each
representing the different errors/warnings that might arise from a given
GHC subsystem. For example (simplified):::

    data RenamerError
        = NotInScope OccName [Name] -- unknown name, suggestions
	| ...


    data TypecheckerError
        = OccursCheck Type Type
	| ...

    ...

We could even split error types further if necessary, making it a
slightly more elaborate hierarchy. The exact shape of the said hierarchy
has yet to be determined, as it will be best informed by staring at the
error generation code that GHC has today. We would also provide an
existential wrapper, to be able to store and more generally treat
uniformly error values of different types:::

    data GHCError where
        GHCError :: (Typeable e, HasErrMsg e) => e -> GHCError

    class HasErrMsg e where
        errorMessage :: e -> ErrMsg

That would give error consumers the ability to simply render the errors
(``HasErrMsg``), but also handle specific error types in a particular
way, by using the ``Typeable`` dictionary to check against ``TypeRep`` s
of interest and then coercing the ``e`` value to the corresponding type.
At no point in time is any parsing of error message involved anymore.

For error producers, unlike an approach with a single hierarchy and no
existential wrapper, this one doesn't introduce delicate import cycles
that require ``.hs-boot`` files to work around. Indeed, there is no need
to import various types from different subsystems in a single place here,
as any part of GHC can define its own error type and wrap it up in
``GHCError`` when we need to store it with other errors, of a potentially
different type.

It is important to note that ``HasErrMsg`` ties this proposal back with
the existing system. Right now, GHC immediately emits error messages
(i.e a textual representation of the errors) and has a lot of code for
rendering all the relevant information (e.g expressions or types)
with some helpful messages. This proposal merely suggests that we keep
this code but call it much later, when GHC's job with the module is done
and the compilation has failed (for errors) or succeeded with warnings,
that we need to report too. GHC would simply keep around all the relevant
information that the textual rendering of those errors requires,
as values of suitably defined algebraic data types, with all the
expressions, types, contexts, suggestions and more stored in fields of
those ADTs.

If necessary, we could define a separate wrapper for warnings and
update the definitions of ``ErrorMessages`` and ``WarningMessages``
given earlier as follows:::

    data GHCWarning w where
        GHCWarning :: (Typeable w, HasErrMsg w) => w -> GHCWarning

    type ErrorMessages   = Bag GHCError
    type WarningMessages = Bag GHCWarning
    type Messages        = (WarningMessages, ErrorMessages) -- as before

(The alternative being to just store ``GHCError`` values in both bags.)

Finally, we would have to update some error reporting infrastructure
to take ``GHCError`` values as arguments instead of ``ErrMsg``. That is
the point at which the actual rendering of error messages would happen,
under this proposal, right before the code that logs the said errors.

A consequence of this is that the ``Messages`` type that GHC API users
consume would now carry error and warning values that they can render
but also inspect, without parsing. A lot of the work would be about
actually moving all the error rendering code away from where we create
errors, and defining suitable types that carry the data around until
it is time to report the errors to the user.

Effect and Interactions
-----------------------

By turning errors into proper values, tooling authors would be able to
get rid of their error parsing code and finally be able to concisely
inspect, render or "decorate" error messages. A few compelling
applications are listed below:

* A REPL front-end might implement color-coded output, choosing a token's
  color by its syntactic class (e.g type constructor, data constructor,
  or identifier), its name (e.g all occurences of ``foldl`` shown in red,
  occurences of ``concat`` shown in blue), or some other criterion
  entirely.

* A REPL front-end or IDE tool might allow users the ability to
  interactively navigate a type in a type error and, for instance, allow
  the user to interactively expand type synonyms, show kind signatures,
  etc.

* An IDE tool might ask GHC to defer expensive analyses typically done
  during error message construction (e.g. `computing valid hole fits
  <https://gitlab.haskell.org/ghc/ghc/issues/16875#note_210045>`_) and instead
  query GHC for the analysis result asynchronously (or even only when
  requested by the user), shrinking the edit/typechecking iteration time.

* An IDE tool might use the suggestions that GHC would embed in error
  values to present automated refactoring options to the user.

Costs and drawbacks
-------------------

With the approach described in this proposal, the different error types
would be scattered accross the different components of the GHC library,
there would therefore not be a single place to look at to see all the
errors. They would have to look for all the places in the code where we
pack some error in a ``GHCError`` wrapper.

Another drawback is that error consumers have to resort to using
``Typeable`` to implement specific behaviours for any given type, instead
of pattern matching on constructors of a large sum type of all possible
errors. Both of those drawbacks come from the dynamic component of the
design put forward in this proposal, which is essential to avoid the
import cycles mentionned earlier. We consider the import cycles to be
a more serious drawback than the dynamic errors, especially given that
the number of concrete error/warning types should be quite reasonable.

The major cost of implementing this proposal is the sheer amount of
refactoring that will be necessary to emit error values and move the
rendering to much later.

Alternatives
------------

As mentionned in the previous section, we considered a similar approach
but where there is no existential wrapper and the errors from different
subsystems are just glued together using sum types. That would have
required importing all subsystems in one place, introducing import cycles
that would be very delicate to handle and incur too expensive a price on
all GHC developers. Another problem (small, in comparison to the
previous one) is that those intermediate "glue" sum types introduce
some maintenance overhead, because of the need to enumerate all types of
errors that GHC can throw, at various points of the module/subsystem
hierarchy, as we make the error types bubble up until we define a
toplevel error ADT that can represent any error that GHC can throw.

On the other end of the spectrum, we could also define one type per
error and just pack them all into an existential wrapper with suitable
dictionaries. That would have the advantage of not requiring any
"glue" but would be very painful for consumers, as they would not
have any convenient way to handle a whole group of errors that are
somewhat related, e.g coming from the same subsystem. This would be
particularly problematic if we end up wanting to introduce a level or
two in the hierarchy between the existential wrapper at the top and a
concrete sum of errors for a given part of GHC (a leaf).

Unresolved Questions
--------------------

We have not fully fleshed out the forest of error types and believe this
is something that will be best done by scanning GHC's code, looking for
functions that emit error messages and trying to adapt them to emit an
existentially-wrapped, subsystem-specific error **value**.

Implementation Plan
-------------------

Well-Typed LLP will implement this proposal with financial support from
Richard Eisenberg, under NSF grant number 1704041.
