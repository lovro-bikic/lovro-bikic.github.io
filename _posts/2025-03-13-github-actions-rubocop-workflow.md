---
layout: post
title: Running RuboCop on GitHub Actions With Cache
---

Here's how to set up a RuboCop workflow on GitHub Actions with caching for faster workflow runs.

This workflow has been battle-tested on a rather active Rails monolith where running RuboCop would take 8 minutes without cache. Fortunately, caching drops that time to ~40 seconds.

The workflow assumes you have a Gemfile that includes the `rubocop` gem.

Finished product first, followed by a detailed breakdown. This can be copy-pasted, but you might need to adjust it to fit your own needs.

{% highlight yaml %}
{% raw %}
# .github/workflows/rubocop.yml
name: RuboCop

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

on:
  pull_request:
  push:
    branches: # Keep one, delete the other
      - master
      - main

jobs:
  rubocop:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    env:
      RUBOCOP_CACHE_ROOT: tmp/rubocop
    steps:
      - name: Git checkout
        uses: actions/checkout@v4
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Prepare RuboCop cache
        uses: actions/cache@v4
        env:
          DEPENDENCIES_HASH: ${{ hashFiles('.ruby-version', '**/.rubocop.yml', 'Gemfile.lock') }}
        with:
          path: ${{ env.RUBOCOP_CACHE_ROOT }}
          key: rubocop-cache-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}
          restore-keys: |
            rubocop-cache-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-
      - name: Run RuboCop
        run: bundle exec rubocop --format github
{% endraw %}
{% endhighlight %}

## Workflow breakdown

There are a few things going on here; let's start from the top.

### Concurrency

{% highlight yaml %}
{% raw %}
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
{% endraw %}
{% endhighlight %}

[Mechanism](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#example-using-concurrency-and-the-default-behavior) to limit in-progress workflow runs to one per branch. In practice, if you push a commit to a branch while a workflow is already running for a previous commit, that run will be cancelled.

### Workflow triggers

{% highlight yaml %}
{% raw %}
on:
  pull_request:
  push:
    branches:
      - master
      - main
{% endraw %}
{% endhighlight %}

Defines events that trigger the workflow.

`pull_request` will run RuboCop when a PR is opened or updated, for example.

`push` will run RuboCop on pushes to the default branch (master/main). This is used to both verify the latest commit passes RuboCop and to update the cache (more on that below).

### Job setup

{% highlight yaml %}
{% raw %}
jobs:
  rubocop:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    env:
      RUBOCOP_CACHE_ROOT: tmp/rubocop
{% endraw %}
{% endhighlight %}

Defines the job and its runner and timeout. Adjust the timeout if necessary, but keep it as low as possible in case a workflow run gets stuck so you don't get billed for wasted minutes ([default timeout is 6 hours](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes), which could cost you a couple $$ if something goes awry and you don't notice it (speaking from experience)).

This part of the workflow also sets up the [`RUBOCOP_CACHE_ROOT`](https://docs.rubocop.org/rubocop/usage/caching.html#cache-path) environment variable to save RuboCop cache in the `tmp/rubocop` folder. This is the folder we'll be saving in GitHub Actions cache later on.

### Repo setup

{% highlight yaml %}
{% raw %}
    steps:
      - name: Git checkout
        uses: actions/checkout@v4
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
{% endraw %}
{% endhighlight %}

[Clones the repo](https://github.com/actions/checkout) and [installs and caches gems](https://github.com/ruby/setup-ruby/).

### RuboCop cache preparation

This is the main dish. The following step uses the [cache action](https://github.com/actions/cache).

{% highlight yaml %}
{% raw %}
      - name: Prepare RuboCop cache
        uses: actions/cache@v4
        env:
          DEPENDENCIES_HASH: ${{ hashFiles('.ruby-version', '**/.rubocop.yml', 'Gemfile.lock') }}
        with:
          path: ${{ env.RUBOCOP_CACHE_ROOT }}
          key: rubocop-cache-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}
          restore-keys: |
            rubocop-cache-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-
{% endraw %}
{% endhighlight %}

The path we're caching is defined in the environment variable `RUBOCOP_CACHE_ROOT` that was set on job-level. Setting this folder explicitly ensures RuboCop cache is located in the same folder that GH Actions caches.

Cache is saved under the key:
{% highlight text %}
{% raw %}
rubocop-cache-
${{ runner.os }}-
${{ env.DEPENDENCIES_HASH }}-
${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }
{% endraw %}
{% endhighlight %}

`{% raw %}${{ runner.os }}{% endraw %}` is added as ["standard" practice](https://github.com/actions/cache?tab=readme-ov-file#example-cache-workflow) to not use a cache if you switch to a runner on a different OS.

`{% raw %}${{ env.DEPENDENCIES_HASH }}{% endraw %}` is an environment variable defined as:
{% highlight text %}
hashFiles('.ruby-version', '**/.rubocop.yml', 'Gemfile.lock')
{% endhighlight %}

[hashFiles](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/evaluate-expressions-in-workflows-and-actions#hashfiles) returns a hash of a given set of files and this particular set aims to follow RuboCop's [cache validity rules](https://docs.rubocop.org/rubocop/usage/caching.html#cache-validity). An existing cache won't be used in a later run if:
<ul>
<li>Ruby version changes (assuming there's a <code>.ruby-version</code> file), or</li>
<li><a href="https://docs.rubocop.org/rubocop/configuration.html#config-file-locations">any</a> <code>.rubocop.yml</code> file changes, or</li>
<li>RuboCop version changes (technically, cache will be invalidated any time Gemfile.lock changes, but this is a pragmatic choice to not overcomplicate the setup).</li>
</ul>

Finally, `{% raw %}${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }{% endraw %}` is added to ensure the default branch has [up-to-date cache](https://github.com/actions/cache/blob/main/tips-and-workarounds.md#update-a-cache) (its cache key ends with [workflow run ID](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#github-context), which changes for each commit). Other branches have a single cache (their cache key ends with `default`).

When [restoring cache](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#matching-a-cache-key), if there's no exact key match, we try to restore the most recent cache that matches the prefix<br />
`{% raw %}rubocop-cache-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-{% endraw %}`

#### How does this setup function in practice?

On all branches, if dependencies (runner OS or `DEPENDENCIES_HASH`) change in a new commit, cache is not restored and RuboCop has to scan the entire codebase from scratch. Consequently, these runs are the slowest ones. When they finish, their RuboCop cache is saved for later use.

On the default branch, each commit creates a new cache entry as means of "updating" the cache. Cache from a previous workflow run is restored if dependencies match, but it will always be saved under a new key. This is necessary because GHA cache is immutable so when there's a cache hit you won't be able to update its contents. If we don't "update" the cache, it will become less and less useful over time as source files change (RuboCop invalidates a file's cache if its contents change).

On branches other than the default, first workflow run will try to restore the most recent cache from [base/default branch](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache). The cache will be saved under a key ending in `default` when the workflow finishes. If dependencies don't change in a later commit, this will be the single cache entry for that branch.

A branch's `default` cache might not be so useful in a later commit if you change a bunch of files, but this setup works okay for most workflows (for example: first commit changes the most files, and later commits update only some files after a PR review, so most of the cache is still utilized).

### Blast-off

Finally, we run RuboCop:

{% highlight yaml %}
{% raw %}
      - name: Run RuboCop
        run: bundle exec rubocop --format github
{% endraw %}
{% endhighlight %}

[GitHub Actions formatter](https://docs.rubocop.org/rubocop/formatters.html#github-actions-formatter) will add nice annotations in the UI if there are any offenses.
<br/>
<br/>
Enjoy!
<br/>
<br/>
#### PS: A note for large codebases

At the time of writing, RuboCop by default [saves max 20k files in cache](https://github.com/rubocop/rubocop/blob/b6678159b618a9274d56fd4a95310fa48f36666c/config/default.yml#L121). If your codebase is larger than that, you'll want to update your configuration so it caches everything in the codebase to ensure optimal workflow times.

Here's how to quickly check the number of files RuboCop will scan:

{% highlight text %}
{% raw %}
$ bundle exec rubocop --list-target-files | wc -l
{% endraw %}
{% endhighlight %}

Then, update `MaxFilesInCache` in your `.rubocop.yml` to a value greater than that.
