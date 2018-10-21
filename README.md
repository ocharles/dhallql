# `dhallql` -- a query language for the web

`dhallql` is a query language for the web - much like GraphQL. This project is
in its infancy, and we're currently in the design phase.

## Proposal

In `dhallql` the main query expression is a function of type `Root -> A` - `Root`
is a type defined by the API, and `A` is the type of data that the client wishes
to retrieve.

`Root` provides access to "root resources", usually as a record of functions,
and `A` is a type built using these root resources to derive a subset of
information that the client wants to work with.

As an example, let's consider the query for the first 100 hundred repositories
of the user `ocharles` on some GitHub. Our GitHub `dhallql` API gives us the
following `Root` type:

``` dhall
    let T = https://dhallql.github.com/Types.dhall

in  { user : { name : Text } → Optional T.User }
```

That is, `Root` is a record type, with one field `user`, who's type is 
`{name : Text} -> Optional T.User`.

This alone gives us the ability to query for users. To get the `ocharles`
`User`, we would send:

``` dhall
λ(root : http://dhallql.github.com/Root.dhall) → root.user { name = "ocharles" }
```

But that's not quite what we want. If we inspect `Types.dhall`, we see that we
can pull information out of `User`s:

``` dhall
    let Repository = { name : Text }

in  let RepositoryConnection = { nodes : List Repository }

in  let User = { repositories : < first : Natural > → RepositoryConnection }

in  { Repository =
        Repository
    , RepositoryConnection =
        RepositoryConnection
    , User =
        User
    }
```

`User` values are records that have a `repositories` function, which gives us a
`RepositoryConnection`, and this has a list of `Repository`s. Thus our query is
just a Dhall expression that turns a `User` into a list of repository names:

``` dhall
    let prelude = https://prelude.dhall-lang.org/package.dhall

in    λ(queries : https://dhallql.github.com/Root.dhall)
    → prelude.`Optional`.map
      T.User
      (List { name : Text })
      (   λ(user : T.User)
        → prelude.`List`.map
          T.Repository
          { name : Text }
          (λ(repo : T.Repository) → { name = repo.name })
          (user.repositories < first = 100 >).nodes
      )
      (root.user { name = "ocharles" })
```

In this more complicated query we're importing the Dhall prelude to get access
to `map` for `List` and `Optional`. We then query 
`root.user { name = "ocharles"}` and map over its return value. If there is such
a user, we call `.repositories` asking for the first 100 repositories. We then
extract just the `name` field of repositories giving us a final query that maps
`ocharles` into a list of repository names.
