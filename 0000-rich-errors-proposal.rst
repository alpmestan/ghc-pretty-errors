Rich error messages from GHC
============================

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

Change GHC's internal error message representation and expose it to tooling consumers.


Motivation
------------
Currently IDE tools (e.g. `Haskell IDE Engine
<https://github.com/haskell/haskell-ide-engine>`_) have to rely on parsing GHC's
human-readable error messages to glean even a basic understanding of the nature
of the error. Not only is this parsing fragile, it also severely limits the
sorts of information the tool can gather about the error. This is unfortunate
since, as GHC ventures further into the territory of dependent types, error
messages grow in complexity and size.

Other languages (e.g. Idris, with their Emacs mode) have `demonstrated
<https://www.youtube.com/watch?v=m7BBCcIDXSg>`_ [1]_ that enriching error
message representations can open the door to useful interactions between the
user, IDE tools, and the compiler. Let's open this door.

Note that this proposal is **not** about changing the default presentation of
error messages produced by the ``ghc`` executable, only the internal
representation. It merely discusses the changes and plumbing necessary
to enable downstream consumers (e.g. REPL shells, editor extensions, and
language servers) to make these changes on their own.

Also see `GHC #8809 <https://gitlab.haskell.org/ghc/ghc/issues/8809>`_.


Proposed Change Specification
-----------------------------

Error messages in GHC are currently represented as simple pretty-printer
documents (note: this is slightly simplified for the sake of discussion),
accompanied by a class to give a common name to the function that turns
all sorts of Haskell values into documents::

    -- | A pretty-printer document.
    data SDoc = ...

    class Outputable a where
      ppr :: a -> SDoc

    -- | An error message.
    type ErrMsg = SDoc


We propose to instead represent errors as dedicated types that
implement the ``IsError`` class that follows.::

    class (Typeable a, Outputable a) => IsError a

We can then uniformly hide the different error types behind an
existential wrapper, packing the right constraints to be able
to process the error.::

    data GhcError where
      GhcError :: IsError a => a -> GhcError

The ``Typeable`` constraint allows error consumers to behave differently
depending on the error('s ``TypeRep``) without having to pack all possible
errors into a giant sum type. The ``Outputable`` constraint on the other
hand guarantees that we can always print an error as a document, no matter
what concrete type is hidden behind a given ``GhcError``.

Below is a sketch of how we could represent the "Expected type: ...,
Actual type: ..." error that we all know and love, as well as the
"Variable not in scope: ..." one.::

    -- expected type/actual type
    data ExpectedActualError = ExpectedActualError
      { expectedType :: Type
      , actualType   :: Type
      }

    instance Outputable ExpectedActualError where
      ppr (ExpectedActualError expected actual) = ...

    instance IsError ExpectedActualError

    -- variable not in scope
    data NotInScope = NotInScope
      { badName               :: OccName
      , suggestedAlternatives :: [Name]
      }

    instance Outputable NotInScope where
      ppr (NotInScope occname alts) = ...

    instance IsError NotInScope

Error consumers (GHC's error reporting mechanism, IDE/tooling
authors) can then decide to simply print the errors, uniformly,
or handle a few special cases in a particular way (e.g look for ``Type`` values
in errors and grab their definitions to allow IDE users to expand types that
appear in error messages, interactively). For GHC's error reporting,
we would consume ``GhcError`` values by displaying the corresponding
error message document, as we currently do, but GHC API users would have the
ability to do more and implement powwerful features like the ones
described in the next section, without having to write fragile error message
parsers.

For error producers, Unlike an approach that would collect all
possible errors under a (quite large) sum type, this one doesn't
introduce delicate import cycles that we would have to work around
with ``.hs-boot`` files. Indeed, the module defining the large
error sum type would have to import quite a few modules from all
parts of GHC to have all the right types in scope, and those modules
would in turn have to import the module with the error sum type, so as
to be able to throw such errors.


Effect and Interactions
-----------------------

By turning errors into proper values (instead of documents), we allow tooling
authors significantly more flexibility in presenting (and automatically
fixing) compile-time errors. We list a few compelling applications below
(roughly in order of complexity):

* A REPL front-end might implement color-coded output, choosing a token's
  color by its syntactic class (e.g. type constructor, data constructor, or
  identifier), its name (e.g. all occurrences of ``foldl`` shown in red,
  occurrences of ``concat`` shown in blue), or some other criterion entirely.

* A REPL front-end or IDE tool might allow users the ability to interactively
  navigate a type in a type error and, for instance, allow the user to
  interactively expand type synonyms, show kind signatures, etc.

* An IDE tool might ask GHC to defer expensive analyses typically done
  during error message construction (e.g. `computing valid hole fits
  <https://gitlab.haskell.org/ghc/ghc/issues/16875#note_210045>`_) and instead
  query GHC for the analysis result asynchronously (or even only when
  requested by the user), shrinking the edit/typechecking iteration time.

* An IDE tool might use the suggestions that GHC would embed in error values
  to present automated refactoring options to the user.


Costs and Drawbacks
-------------------

With the approach described in this proposal, we don't have a single place to
look at to see all the errors that can be thrown, since one of the advantages
of this approach is that all GHC subsystems can define and use their own errors
without requiring any other part of GHC to know about it. This will require
error consumers to grep for ``IsError`` instances or build and browse the
haddock documentation, to get the aforementionned big picture.

Another drawback of this approach is that error consumers have to use
``Typeable`` to implement some specific behaviour that depends on the error
value to be reported (such as the ones described in the previous section), while
an approach using a large sum type with one constructor per error that GHC can
throw would do this by pattern matching on the suitable constructors for those
errors that require some special treatment. It is n


Alternatives
------------

We explored a few alternative points in the design space, and wrote a
summary of the pros/cons of all approaches (including the one we are
proposing here) in
[this document](https://github.com/bgamari/ghc-pretty-errors/blob/master/overview.mkd).


Unresolved Questions
--------------------

We need to put some thoughts into the design of a few helper types and functions
to capture common error patterns, such as attaching suggestions to error
messages. This will however be best informed by looking at the many errors
thrown by GHC and by thinking about how we would like to represent them.


Implementation Plan
-------------------

Well-Typed LLP will implement this proposal with financial support from
Richard Eisenberg.
