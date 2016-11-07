# git-pisect - Parallel regression finder

# Usage

# What's with the name?

It's a really crappy portmanteau of "git parallel bisect".

# What? How can you run bisect in parallel?

I know, I know - you need to finish one round of tests to be able
to decide the next commit to test, right?  This is why the name is
crappy - it should really be something like "git-pnsect" (for parallel
n-sect) or "git-parallel-regression-finder", but git-pisect is just
so catchy.

# How does it work?

## Regular Bisect

Well, let's look at your typical range of Git commits:

    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

Let's say that `HEAD~15` passes, but `HEAD` fails.  So if you run `git bisect start HEAD HEAD~15`, git starts testing at the halfway point:

                                                                             ↓
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

If the regression isn't present at the current point (`HEAD~8` here), we know that it must have been introduced in a later commit, so we pick a new point
halfway between the last known good commit and the first known bad commit, like so:

                                                                                                                 ↓
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

Assuming that `HEAD~4` fails, we can narrow the range even further:

                                                                                               ↓
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

If the test fails at `HEAD~6`, we know that either it or `HEAD~7` is the culprit, so we do a final test with the files from `HEAD~7`.  Let's say that
`HEAD~7` is indeed the culprit for this next part.

## Parallel "bisect"

XXX Talk about `git-bisect run`

So let's say you're doing this on a repository that has a test suite that takes about a minute to run, is safely parallelizable, and doesn't implement any
sort of parallelization itself.  Since most of our machines these days have multiple cores, running these tests on just a single core seems like a waste
of time.  What if we are using a machine with four cores - can we use all of them?  Yes!  Let's look at that example from before:

                                       ↓                            ↓                          ↓                          ↓
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

Running four tests in parallel, we can move on to a smaller range of commits under consideration much more quickly.  After the step, I demostrated above,
here's the next one:

                                                                             ↓        ↓
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────┐
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    │ HEAD~15 │ HEAD~14 │ HEAD~13 │ HEAD~12 │ HEAD~11 │ HEAD~10 │ HEAD~9 │ HEAD~8 │ HEAD~7 │ HEAD~6 │ HEAD~5 │ HEAD~4 │ HEAD~3 │ HEAD~2 │ HEAD~1 │ HEAD │
    │         │         │         │         │         │         │        │        │        │        │        │        │        │        │        │      │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────┘

So instead of taking four steps (ie. four minutes) as in the serial bisect, we only take two!

# Metrics

TODO

# Dependencies

TODO

# Caveats

TODO
