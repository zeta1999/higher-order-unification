## An Explanation of Unification.hs

In order to make this code useful to others, I would like to take the
time to explain exactly how it works. In this file, we will go through
the unification algorithm being used and how it is implemented in the
code.

### The Problem

Before beginning, it's worth clarifying the problem that we're
attempting to solve with this code, namely, what is higher order
unification? The simple answer is that we want to take two terms with
"holes" in them, called metavariables. We then want to figure out how
to replace those metavariables with programs so that the two terms,
once fully filled in, evaluate to the same term. Our language will
contain the following constructs,

 1. Variables
 2. Functions (lambdas) and correspondingly application
 3. Metavariables
 4. Function types, in the typical style for a dependently typed
 language: pi types
 5. Universe types

This was originally designed to be part of a typechecker for a
particular dependently typed language, hence the pi types and
universes, but they can safely be ignored and treated as particular
constants.

The main issue is, that it's actually undecidable to do this in the
general case. It's therefore only possible to implement a semidecision
procedure that performs relatively well in practice. By a semidecision
procedure, I mean an algorithm that will terminate with a solution
when possible and reject only some of the time. This procedure is
called Huet's algorithm and it's essentially a refinement of the
following algorithm

 1. Generate a solution
 2. Test it
 3. If the solution was correct, stop
 4. Else, go to 1

This is not exactly the most sophisticated algorithm but it does have
the benefit of being obviously a correct semidecision procedure for
our problem. The idea with Huet's algorithm is to gradually produce a
solution and to only produce solutions that are at least not
obviously wrong. By doing this, we drastically cut down the search
space and produce answers reasonably quickly.

### The Set Up

To begin with, we introduce the tools that we will need to even code
up the unification algorithm. The first critical point is how we
define the language we're unifying in the first place. I will
represent terms using the so-called "locally nameless" approach. This
means that we use
[de Bruijn](https://en.wikipedia.org/wiki/De_Bruijn_index) to
represent bound variables. However, for free variables we will
generate globally unique identifiers to simplify the process of
carrying around contexts or the like. This does mean that our AST has
two different constructors for variables, free and bound.

``` haskell
    type Id = Int
    type Index = Int
    data Term = FreeVar Id
              | LocalVar Index
              | MetaVar Id
              | Uni
              | Ap Term Term
              | Lam Term
              | Pi Term Term
              deriving (Eq, Show, Ord)
```

Since we're using de Bruijn indices, we also need to define a crucial
helper function called `raise :: Int -> Term -> Term`. This raises all
the variables wrapped in a `LocalVar` constructor up by `i`. This is
done by recursing over the inputted term.

``` haskell
    raise :: Int -> Term -> Term
    raise = go 0
      where go lower i t = case t of
              FreeVar i -> FreeVar i
              LocalVar j -> if i > lower then LocalVar (i + j) else LocalVar j
              MetaVar i -> MetaVar i
              Uni -> Uni
              Ap l r -> go lower i l `Ap` go lower i r
              Lam body -> Lam (go (lower + 1) i body)
              Pi tp body -> Pi (go lower i tp) (go (lower + 1) i body)
```

Using this, we can define substitution on terms. This will be useful
later on directly. For this, we first define the notion of replacing a
de Bruijn variable with a term.

``` haskell
    subst :: Term -> Int -> Term -> Term
    subst new i t = case t of
      FreeVar i -> FreeVar i
      LocalVar j -> case compare j i of
        LT -> LocalVar j
        EQ -> new
        GT -> LocalVar (j - 1)
      MetaVar i -> MetaVar i
      Uni -> Uni
      Ap l r -> subst new i l `Ap` subst new i r
      Lam body -> Lam (subst (raise 1 new) (i + 1) body)
      Pi tp body -> Pi (subst new i tp) (subst (raise 1 new) (i + 1) body)
```

Notice that we have used `raise` to escape `new` as we go under
binders to avoid capturing variables. Similarly, since we're removing
a binding level, if we have any de Bruijn variables that refer to a
binding site outside of the one we're working with we have to lower it
to compensate. That is the reason for the line `GT -> LocalVar (j - 1)`.
Apart from these two complications, substitution is just hunting for
all occurrences of `LocalVar i` and replacing it with `new`. However,
we also have this metavariables so it makes sense that we have a
notion of substitution for these as well. It's simpler than the above
substitution function because we don't have to worry about lowering
variables that might be affected by deleting a metavariable binding
since we're using globally unique identifiers for them.

``` haskell
    substMV :: Term -> Id -> Term -> Term
    substMV new i t = case t of
      FreeVar i -> FreeVar i
      LocalVar i -> LocalVar i
      MetaVar j -> if i == j then new else MetaVar j
      Uni -> Uni
      Ap l r -> substMV new i l `Ap` substMV new i r
      Lam body -> Lam (substMV (raise 1 new) i body)
      Pi tp body -> Pi (substMV new i tp) (substMV (raise 1 new) i body)
```

Now there are only a few more utility functions left before we can get
to the actual unification. We need a function to gather all of the
metavariables in a term. For this we use a `Set` from `containers` and
just fold over the structure of the term.

``` haskell
    metavars :: Term -> S.Set Id
    metavars t = case t of
      FreeVar i -> S.empty
      LocalVar i -> S.empty
      MetaVar j -> S.singleton j
      Uni -> S.empty
      Ap l r -> metavars l <> metavars r
      Lam body -> metavars body
      Pi tp body -> metavars tp <> metavars body
```

Another useful function will be necessary for enforcing the condition
that we only unify metavariables with closed terms (no capturing). In
order to handle this, we will need to check that a given term is
closed. This is as simple as looking to see if it mentions the
`FreeVar` constructor since `LocalVar` is used for only bound
variables by invariant.

``` haskell
    isClosed :: Term -> Bool
    isClosed t = case t of
      FreeVar i -> False
      LocalVar i -> True
      MetaVar j -> True
      Uni -> True
      Ap l r -> isClosed l && isClosed r
      Lam body -> isClosed body
      Pi tp body -> isClosed tp && isClosed body
```

The last complicated utility function is `reduce`. This is actually
just a simple interpreter for the language we defined earlier. It
essentially repeatedly searches for `Ap (Lam ...) ...` and when it
finds such an occurrence substitutes the argument into the body of the
function as one might expect. I have made this function reduce
everywhere because it seems to provide a significant performance
improvement in many cases.

``` haskell
    reduce :: Term -> Term
    reduce t = case t of
      FreeVar i -> FreeVar i
      LocalVar j -> LocalVar j
      MetaVar i -> MetaVar i
      Uni -> Uni
      Ap l r -> case reduce l of
        Lam body -> reduce (subst r 0 body)
        l' -> Ap l' (reduce r)
      Lam body -> Lam (reduce body)
      Pi tp body -> Pi (reduce tp) (reduce body)
```

The remaining utility funcitons are simply checks and manipulations
that we will frequently need on terms. We have a function which checks
whether a term is of the form `M e1 e2 e3 ...` for some metavariable
`M`; such terms are said to be stuck.

``` haskell
    isStuck :: Term -> Bool
    isStuck MetaVar {} = True
    isStuck (Ap f _) = isStuck f
    isStuck _ = False
```

The remaining utility functions simply convert telescopes of
applications, `f a1 a2 a3 ...`, into an function and a list of
arguments, `(f, [a1 ... an])` and then we have a function to put
things back again.

``` haskell
    peelApTelescope :: Term -> (Term, [Term])
    peelApTelescope t = go t []
      where go (Ap f r) rest = go f (r : rest)
            go t rest = (t, rest)

    applyApTelescope :: Term -> [Term] -> Term
    applyApTelescope = foldl' Ap
```

We are now in a position to turn to implementing the actual
unification algorithm with all of our utilities in hand.

### The Unification Algorithm

There are really only two key functions in implementing the
unification algorithm. We can either take an existing constraint and
simplify it, or take a constraint and produce a list of partial
solutions, at least one of which is correct if the constraint is
solvable. The first function is remarkably similar to the first-order
case of unification, we essentially take a constraint and produce a
set of constraints which are equivalent to the original one. For
instance, if our constraint that we're trying to solve is

``` haskell
    FreeVar 0 `Ap` E === FreeVar 0 `Ap` E'
```

It's easy to see that we might as well solve constraint `E === E'`
which is strictly simpler. This is what the function `simplify`
does. It has the type

``` haskell
    simplify :: Constraint -> UnifyM (S.Set Constraint)
```

In order to work with generating fresh metavariables and (later)
backtracking, we use the monad `UnifyM`. This is defined, as is
`Constraint`, as a type synonym

``` haskell
    type UnifyM = LogicT (Gen Id)
    type Constraint = (Term, Term)
```

Here we are using the package
[logict](https://hackage.haskell.org/package/logict) to provide
backtracking. My tutorial of this package can be found
[here](https://jozefg.bitbucket.io/posts/2014-07-10-reading-logict.html). We
are also using a package a threw together a few years ago called
[`monad-gen`](https://hackage.haskell.org/package/monad-gen), it just
provides a simple monad for generating fresh values. The sort of thing
that I always end up needing in compilers. Without further-ado, let's
start going through the cases for `simplify`. Each one of which
corresponds to a simplifying move we are allowed to make on a
constraint, ordered in terms of priority.

``` haskell
    simplify (t1, t2)
      | t1 == t2 = return S.empty
```

We start out with a nice and simple case, if the two terms of the
constraint are literally identical, we have no further goals. Next we
have two cases integrating reduction. If either term is reducible at
all we reduce it and try to simplify the remaining goals.

``` haskell
      | reduce t1 /= t1 = simplify (reduce t1, t2)
      | reduce t2 /= t2 = simplify (t1, reduce t2)
```

This is how we integrate the fact that our unification is modulo
reduction (we allow two terms to unify if they reduce to the same
thing). Next comes the cases that are a little more sophisticated and
correspond more closely to our original motivating example. If our two
terms are a several things applied to free variables, we know the
following

 1. The free variables have to be the same
 2. All of the arguments must unify

This is captured by the following branch of simplify.

``` haskell
      | (FreeVar i, cxt) <- peelApTelescope t1,
        (FreeVar j, cxt') <- peelApTelescope t2 = do
          guard (i == j && length cxt == length cxt')
          fold <$> mapM simplify (zip cxt cxt')
```

This code just codifies the procedure that we have informally sketched
above. If we're trying to unify `A a1 ... an` and `B b1 ... bm` for
two free variables `A` and `B` then we must have `A = B` and `n = m`
since we have to find a solution that works for any `A` and any
`B`. Finally, we then just need to unify `ai` with `bi`. The next two
cases are congruence type rules. We basically just produce new
constraints saying that `Lam e === Lam e'` if and only if `e ===
e'`. There is a small amount of bookkeeping done to make sure that
free variables are correctly represented by a globally unique
`FreeVar i`. The same thing is done for `Pi` except, since `Pi`s are
annotated with a type we also add a constraint for these types as well.

``` haskell
      | Lam body1 <- t1,
        Lam body2 <- t2 = do
          v <- FreeVar <$> lift gen
          return $ S.singleton (subst v 0 body1, subst v 0 body2)
      | Pi tp1 body1 <- t1,
        Pi tp2 body2 <- t2 = do
          v <- FreeVar <$> lift gen
          return $ S.fromList
            [(subst v 0 body1, subst v 0 body2),
             (tp1, tp2)]
```

The final case is to decide whether or not the constraint is "stuck"
on a metavariable, in which case we'll need to guess a solution for a
metavariable or whether the constraint is just impossible. If neither
constraint is stuck, we fail using `mzero` and if we're stuck then we
just return the inputted constraint since we can make it no simpler.

``` haskell
      | otherwise =
        if isStuck t1 || isStuck t2 then return $ S.singleton (t1, t2) else mzero
```

Now we turn to the most complicated part of the algorithm, where we
actual try and produce possible and partial solutions for our
unification constraints. The basic idea is to work with constraints of
the form

``` haskell
    M a1 a2 ... an = A b1 b2 ... bm
```

where `M` is a metavariable and `A` is a some term, probably a free
variable. These are called flex-rigid equations because one half is
flexible, a metavariable, while one half is rigid. The first part of
this code is to extract the relevant pieces of information from the
constraint. Therefore, the code roughly looks like

``` haskell
    tryFlexRigid :: Constraint -> [UnifyM [Subst]]
    tryFlexRigid (t1, t2)
      | (MetaVar i, cxt1) <- peelApTelescope t1,
        (stuckTerm, cxt2) <- peelApTelescope t2,
        not (i `S.member` metavars t2) = error "TODO"
      | (MetaVar i, cxt1) <- peelApTelescope t2,
        (stuckTerm, cxt2) <- peelApTelescope t1,
        not (i `S.member` metavars t1) = error "TODO"
      | otherwise = []
```

This simply uses `peelApTelescope` to extract the 4 components `M`,
`(a1 ... an)`, `A` and `(b1 ... bm)`. The resulting type is "morally"
supposed to be `[Subst]` but for technical reasons we need to
`[UnifyM [Subst]]` because we need to generate metavariables for the
substitutions. There are exactly 2 forms that `M` may take

 - `M = λ x1. ... λ xn. xi (M1 x1 ... xn) ... (Mr x1 ... xn)`
 - `M = λ x1. ... λ xn. A (M1 x1 ... xn) ... (Mr x1 ... xn)`
   (if `A` is closed)

These are the only two forms that `M` can take because if `M` is any
other constant or free variable than it would immediately
contradictory, `M` couldn't possibly unify with `A b1 ... bm` as we
need it to. Therefore, `tryFlexRigid` will produce a list of such
substitutions (mod effects) replacing `M` with both of these. Since we
don't know how many subterms we must apply to `xi` or `A` this will be
an infinitely long list. More on this complication will
follow. Therefore, we can replace `error "TODO"` with

``` haskell
    type Subst = M.Map Id Term

    tryFlexRigid :: Constraint -> [UnifyM [Subst]]
    tryFlexRigid (t1, t2)
      | (MetaVar i, cxt1) <- peelApTelescope t1,
        (stuckTerm, cxt2) <- peelApTelescope t2,
        not (i `S.member` metavars t2) = proj (length cxt1) i stuckTerm 0
      | (MetaVar i, cxt1) <- peelApTelescope t2,
        (stuckTerm, cxt2) <- peelApTelescope t1,
        not (i `S.member` metavars t1) = proj (length cxt1) i stuckTerm 0
      | otherwise = []
```

Here `proj` generates the list of substitutions. It's arguments are

 1. The number of bound variables
 2. The metavariable we're trying to find substitutions for
 3. The term `A` that we may use to construct a substitution for `M`
 4. The number of subterms to generate (this will be incremented in
    the recursive call)

It's defined just as

``` haskell
    proj bvars mv f nargs =
      generateSubst bvars mv f nargs : proj bvars mv f (nargs + 1)
```

Now the work is done in the actual function
`generateSubst :: Int -> Id -> Term -> Int -> UnifyM [Subst]`. We have
already explained the behavior of `generateSubst`, it's just going to
create all possible substitutions of the form described above. There
is little more to say than to just show the code.

``` haskell
    generateSubst bvars mv f nargs = do
      let mkLam tm = foldr ($) tm (replicate bvars Lam)
      let saturateMV tm = foldl' Ap tm (map LocalVar [0..bvars - 1])
      let mkSubst = M.singleton mv
      args <- map saturateMV . map MetaVar <$> replicateM nargs (lift gen)
      return [mkSubst . mkLam $ applyApTelescope t args
             | t <- map LocalVar [0..bvars - 1] ++
                    if isClosed f then [f] else []]
```

All that is left to do is to tie these two functions together in to
try and produce a solution in general. One small caveat is that we
need a few simple functions for working with substitutions. One to
take a `Subst` and perform all the indicated replacements on a term
and one to take two substitutions and perform a disjoint merge on
them.

``` haskell
    manySubst :: Subst -> Term -> Term
    manySubst s t = M.foldrWithKey (\mv sol t -> substMV sol mv t) t s

    (<+>) :: Subst -> Subst -> Subst
    s1 <+> s2 | not (M.null (M.intersection s1 s2)) = error "Impossible"
    s1 <+> s2 = M.union (manySubst s1 <$> s2) s1
```

Now our main function, `unify` will take the current substitution and
a set of constraints and produce a solution substitution and a set of
flex-flex equations. These are equations of the form
`M a1 ... an = M' b1 ... bn`. It is provable that so called flex-flex
equations are always solvable (cf Huet's lemma) but solving them in a
canonical way is impossible so we instead produce the solution "up to"
flex-flex equations and let the user deal with the ambiguity however
they choose. For example, such an equation in resulting from Agda's
unification algorithm will produce the error "unresolved
metavariables" because the metavariable is not canonically
determined. Therefore, our main algorithm proceeds in the following
steps

1. Apply the given substitution to all our constraints.
2. Simplify the set of constraints to remove any obvious ones.
3. Separate flex-flex equations from flex-rigid ones.
4. Pick a flex-rigid equation at random, if there are none, we're
   done.
5. Use `tryFlexRigid` to get a list of possible solutions
6. Try each solution and attempt to unify the remaining constraints,
   backtracking if we get stuck

In order to implement 2, we define a function which is simply the
"closure" of `simplify` and applies it until there is no more
simplification to be done.

``` haskell
    repeatedlySimplify :: S.Set Constraint -> UnifyM (S.Set Constraint)
    repeatedlySimplify cs = do
      cs' <- fold <$> traverse simplify (S.toList cs)
      if cs' == cs then return cs else repeatedlySimplify cs'
```

Apart from this, the main routine four unification is quite
declarative

``` haskell
    unify :: Subst -> S.Set Constraint -> UnifyM (Subst, S.Set Constraint)
    unify s cs = do
      let cs' = applySubst s cs
      cs'' <- repeatedlySimplify cs'
      let (flexflexes, flexrigids) = S.partition flexflex cs''
      if S.null flexrigids
        then return (s, flexflexes)
        else do
          let psubsts = tryFlexRigid (S.findMax flexrigids)
          trySubsts psubsts (flexrigids <> flexflexes)
```

The first line implements step 1, using
`applySubst :: Subst -> S.Set Constraint -> S.Set Constraint` to apply
our substitution. The next line simplifies the constraints so we're
left with flex-flex or flex-rigid constraints. After this, we can
partition the constraints into these two classes. From here, we simply
implement steps 4-6 making use of the helper function `trySubst`

``` haskell
    trySubsts :: [UnifyM [Subst]] -> S.Set Constraint -> UnifyM (Subst,S.Set Constraint)
```

This function takes care of peeling out each substitution and applying
it to the constraints we have lying around. In order to cope with the
fact that all of these are potentially infinite and we need to fairly
search the resulting space, we make use of
`interleave :: m a -> m a -> m a` from `logict`. It's essentially
equivalent to `mplus` from the list monad but search fairly in the
case of infinite lists. This takes care of handling backtracking in a
seamless and mostly invisible way, Haskell is fun sometimes! The code
for implementing this is essentially just `interleave`-ing all the
recursive calls to `unify` that we need to make using `mzero`,
failure, for when we've run out of substitutions to try.

``` haskell
    trySubsts [] cs = mzero
    trySubsts (mss : psubsts) cs = do
      ss <- mss
      let these = foldr interleave mzero [unify (newS <+> s) cs | newS <- ss]
      let those = trySubsts psubsts cs
      these `interleave` those
```

Putting all of this code together, we have completed a higher-order
unificaiton algorithm! To make a top-level function to play with, we
add a driver function which runs `unify` and strips out all of the
monads of `UnifyM`

``` haskell
    driver :: Constraint -> Maybe (Subst, S.Set Constraint)
    driver = listToMaybe . runGenFrom 100 . observeAllT . unify M.empty . S.singleton
```
