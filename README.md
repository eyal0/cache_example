# cache_example

**tl;dr**: Decide during the job whether or not you want to save a
cache.

## Background

You want to use [caching](https://github.com/actions/cache) in Github
CI to speed up your CI.  One way is to use the same key each time,
like this:

```yaml
      - name: Cache $HOME/.local directory
        uses: actions/cache@v3
        with:
          path: '$HOME/.local'
          key: home-local
```

(`$HOME/.local` is a common place to store programs that you have
installed yourself.)

This will cache the contents of the `$HOME/.local` directory.  The
first time that you run the job it won't do anything because the cache
doesn't exist.  If the job succeeds then it will cache the contents of
the directory.

The next time that you run, it will retrieve that cache.  And because
the retrieval worked, it won't cache it again.

## The problem

What if a cache was retrieved but you want to overwrite it anyway, but
only if the online version is new?

## The workaround (that doesn't quite work).

One way is a
[workaround](https://github.com/actions/cache/blob/main/workarounds.md#update-a-cache)
to force the cache to be written always.  For this, you can put a
unique value in the `key`:

```yaml
      - name: Cache $HOME/.local directory
        uses: actions/cache@v3
        with:
          path: '$HOME/.local'
          key: home-local-${{ github.run_id }}-${{ github.run_attempt }}
          restore-keys: |
            home-local-
```

The `run_id` and `run_attempt` pair is unique for each run so the key
will never match.  The restore-key will match the most recent caches
using a [prefix
match](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) so you'll get the latest.  And the cache will always save because the `key` wasn't matched.

This solution will thrash your cache because it will use up all the quota.

## The solution

We'd like to have the CI decide, during execution, if the contents of
the cache need updating.  The CI could run code to check the version
number installed and compare against the version number online.  And
only if there is a mismatch, download the new version, recompile, and
install it.  And the cache should be updated only if this happened.
Otherwise, leave the old cache.

The solution is to designate an environment variable, for example
called `UPDATE_CACHE`.  When it is set to `true`, the cache it always
updated at the end of a successful job.  And when `false`, the cache
is never updated.  Otherwise, do the usual behavior.

Now we can add a version number check in our CI job and dynamically
decide if we want to save the cache or not.

Here's a
[demonstration](https://github.com/eyal0/cache_example/blob/main/.github/workflows/ci.yml)
of code that will only compile, install and cache a new
[gerbv](https://gerbv.github.io/) only if the online one is new.

## Caveats

### Overwriting a cache

Due to the code on the caching server, [it is not possible to
overwrite a
cache](https://github.com/actions/cache/pull/498#issuecomment-808337154).
I tested this by looking at the caching code and trying to contact the
server directly.  It won't work so we still need to do the workaround
trick.

### Setting the environment variable

Set the environment variable in your CI job like this:

```yaml
      run: echo "UPDATE_CACHE=false" >> $GITHUB_ENV
```

Don't use this:

```yaml
      run: echo "export UPDATE_CACHE=false" >> ~/.bashrc
```

The latter won't work, I think because the shell running a `uses` will
not `source .bashrc`.  The environment variables set that way will not
be available.
