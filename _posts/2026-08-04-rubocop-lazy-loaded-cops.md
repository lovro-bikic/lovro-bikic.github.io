---
layout: post
title: "RuboCop Lazy Loads Cops Now"
excerpt: "Recap of an initiative that improved RuboCop startup times."
---

A mildly exciting performance refactor has landed in RuboCop [v1.89.0](https://github.com/rubocop/rubocop/releases/tag/v1.89.0): [lazy-loaded cops](https://github.com/rubocop/rubocop/issues/14983)! In short:

- RuboCop used to require all 600+ cops at startup,
- it now autoloads cops instead of requiring them,
- startup time has improved noticeably (10% up to 25%, depending on use case),
- plugin maintainers should migrate to the new API for everyone to benefit.

The rest of the post is this summary expanded with details.

## Requires Just Kept Growing

Some context first. Early versions of RuboCop didn't have many cops. For example, v0.3.0 (released in 2013) had only 31; departments didn't even exist as a concept. Back then, it was fine to [require all cops in `lib/rubocop.rb`](https://github.com/rubocop/rubocop/blob/c28cfea0264059e02d9865e9a0f670737f9b5fa1/lib/rubocop.rb).

But RuboCop kept growing. By v1.0.0, the number of cops exceeded 400. They were still required the same way; as a result, `lib/rubocop.rb` was [pretty long](https://github.com/rubocop/rubocop/blob/v1.0.0/lib/rubocop.rb).

Today, there are [more than 600 cops](https://github.com/rubocop/rubocop/blob/v1.88.2/lib/rubocop.rb). And with plugins included (e.g. `rubocop-rails`, `rubocop-rspec`, `rubocop-performance`, etc.), that number is even higher.

Requiring so many cops wouldn't be an issue were we to actually use them all in each RuboCop run. But we don't:

- most RuboCop configs enable only a subset of cops (e.g. [Standard currently enables ~60%](https://github.com/standardrb/standard/blob/v1.56.0/config/base.yml) of all cops),
- you can run RuboCop with a single department (e.g. `bundle exec rubocop -x` runs only Layout cops),
- or even a single cop (e.g. `bundle exec rubocop --only Style/HashSlice`).

Now a `require` in itself is not necessarily a slow operation, but hundreds of requires do add up:

<img src="/images/rubocop_before.png" width="100%" />

This is a performance profile (created with [Vernier](https://github.com/jhawthorn/vernier)) of a RuboCop run with a single cop on just one file. The many "icicles" in the red box are all the cops being required. Even if you can't understand this chart very well, it's quite obvious that the highlighted part sticks out.

So, at this scale, requiring all cops at startup is wasteful. To optimize execution times, a natural solution is to load a cop file only when the cop is going to be used, i.e. load it lazily.

## Some Technical Details

Ruby already has an API to lazy load a class/module: [Kernel#autoload](https://docs.ruby-lang.org/en/4.0/Kernel.html#method-i-autoload).

You might think that to lazy load cops, you just have to replace `require` with `autoload`. If it were that easy, this would be the shortest section in the post.

Here's the thing. All cops inherit from the `RuboCop::Cop::Base` class, which defines a class method `inherited`:

{% highlight ruby %}
{% raw %}
# lib/rubocop/cop/base.rb
def self.inherited(subclass)
  # ... truncated ...
  Registry.global.enlist(subclass)
end
{% endraw %}
{% endhighlight %}

It's a callback method ["invoked whenever a subclass of the current class is created."](https://docs.ruby-lang.org/en/4.0/Class.html#method-i-inherited) When you create your own cop class, this method will be called.

[`RuboCop::Cop::Registry`](https://github.com/rubocop/rubocop/blob/dcbddbf07d36f316c3099397fb548e83ceb389d0/lib/rubocop/cop/registry.rb) is a class that saves references to cop classes. It's used in various places in RuboCop, which I'll get to in a bit. You create an instance of a registry and start adding cops to it (with `#enlist`).

`Registry.global` returns a global instance of the registry. You can, for example, call `RuboCop::Cop::Registry.global.cops` in the console to get a list of all cops.

So this callback adds a cop to the global registry whenever a cop class is created.

As was said, the registry is used in various places in RuboCop. Notably, [`Runner`](https://github.com/rubocop/rubocop/blob/dcbddbf07d36f316c3099397fb548e83ceb389d0/lib/rubocop/runner.rb) (which processes files for offenses) uses it to [select cops to run based on `.rubocop.yml` and provided CLI options](https://github.com/rubocop/rubocop/blob/ceb63776c63cefc7537f4d117b655daa4b2f0ce4/lib/rubocop/runner.rb#L485-L502). If we don't load cop classes, they won't be added to the registry, and then `Runner` won't work properly.

The lazy-loading initiative solved this problem in two steps:

1. `Registry` has been refactored to support enlisting cops by stringified constant name. When the cop class is actually needed, [`Kernel.const_get` is used to resolve the class](https://github.com/rubocop/rubocop/blob/dcbddbf07d36f316c3099397fb548e83ceb389d0/lib/rubocop/cop/registry.rb#L414).
2. A new method [`register_cop`](https://github.com/rubocop/rubocop/blob/dcbddbf07d36f316c3099397fb548e83ceb389d0/lib/rubocop/cop/lazy_loader.rb#L30) has been added to the API. It `autoload`s a cop class and adds the cop to the `Registry` as a stringified constant name.

Here's an example:

{% highlight ruby %}
{% raw %}
# lib/rubocop/cop/lint.rb
module RuboCop
  module Cop
    module Lint
      extend LazyLoader

      register_cop :AmbiguousAssignment, "#{__dir__}/lint/ambiguous_assignment"
      # other cops...
    end
  end
end
{% endraw %}
{% endhighlight %}

When `lib/rubocop/cop/lint.rb` is required, `register_cop` is called, which registers the cop `Lint/AmbiguousAssignment` under constant name `"RuboCop::Cop::Lint::AmbiguousAssignment"` in the global registry, and also `autoload`s the class (cop name and class name are the same).

Later, `Runner` accesses the cop from the registry, which calls `Kernel.const_get` to get the class, and `autoload` magic returns the `AmbiguousAssignment` class.

That is the gist of it. For in-depth details, please check out the PRs referenced in [this issue](https://github.com/rubocop/rubocop/issues/14983).

## Show Me the Numbers

Before the numbers, here's the Vernier profile on lazy-loading RuboCop for the same scenario as above (single cop on just one file):

<img src="/images/rubocop_after.png" width="100%" />

This is what we were aiming for, reducing the number of icicles.

Now for numbers. As was said above, speedup depends on the number of running cops. For simplicity, [I've benchmarked](https://gist.github.com/lovro-bikic/f3236a121501243d254a84e3d7db739f) all runs on a single Ruby file (multiple files naturally take longer) with caching disabled. All _before_ and _after_ runs have been repeated 30 times. Each run will display average execution time with standard deviation.

I'll show you three RuboCop scenarios, in the order of most to fewest running cops:

1. `rubocop` ([Standard](https://github.com/standardrb/standard) config)
2. `rubocop --only [department]`
3. `rubocop --only [cop]`

_After_ runs are on commit [e9defb6](https://github.com/rubocop/rubocop/commit/e9defb651dff92627ac49ddca76211011c7cf08c) (when lazy-loading was added); _before_ runs are on [8dc65e7](https://github.com/rubocop/rubocop/commit/8dc65e7705064ff404847fa98778df50efbb334b) (the one before lazy-loading).

### 1. `rubocop` (Standard config)

For this setup, I configured RuboCop with [Standard base config](https://github.com/standardrb/standard/blob/v1.56.0/config/base.yml) (353 enabled cops) and ran `bundle exec rubocop --cache false`.

Before: 1.650 ± 0.022 s<br />
After: 1.473 ± 0.018 s

Execution time change: **-10.73%**

### 2. `rubocop --only [department]`

Similar setup to above (no Standard installed). I chose the Layout department. Around 100 cops ran.

Before: 1.578 ± 0.040 s<br />
After: 1.280 ± 0.061 s

Execution time change: **-18.88%**

### 3. `rubocop --only [cop]`

Similar setup to above. I chose the `Style/HashSlice` cop.

Before: 1.554 ± 0.017 s<br />
After: 1.138 ± 0.019 s

Execution time change: **-26.77%**

### Interpretation

While the first _before_ run is the slowest one, all _before_ runs share similar execution times. This comes down to the fact that for a single file, most of the execution time is spent on startup rather than processing the file for offenses.

_After_ runs have different times depending on the number of running cops. Fewer cops correlate with better times. The logical conclusion is that the best RuboCop times are achieved with all cops disabled.

Standard is a very popular config, so I would say that ~10% improvement in startup time is the realistic win of this initiative. But please note that for many processed files, the improvement won't be as visible because the time spent processing files will overshadow startup times.

## How To Migrate to the New API

Lazy-loading was only added to RuboCop itself, but plugins will still `require` cops unless they too migrate to the new API. [RuboCop's documentation has all the details on lazy-loading](https://github.com/rubocop/rubocop/blob/v1.89.0/docs/modules/ROOT/pages/development.adoc#cop-lazy-loading), so I'll just briefly demonstrate how to migrate away from `require`.

Whereas before you'd have a file to require all cops:

{% highlight ruby %}
{% raw %}
# lib/rubocop.rb
require_relative 'rubocop/cop/bundler/duplicated_gem'
require_relative 'rubocop/cop/bundler/duplicated_group'
require_relative 'rubocop/cop/bundler/gem_comment'
require_relative 'rubocop/cop/bundler/gem_filename'
require_relative 'rubocop/cop/bundler/gem_version'
require_relative 'rubocop/cop/bundler/insecure_protocol_source'
require_relative 'rubocop/cop/bundler/ordered_gems'
{% endraw %}
{% endhighlight %}

You can now use the `register_cop` method to register the cops for lazy-loading in the department namespace:

{% highlight ruby %}
{% raw %}
# lib/rubocop/cop/bundler.rb
module RuboCop
  module Cop
    module Bundler
      extend LazyLoader

      register_cop :DuplicatedGem, "#{__dir__}/bundler/duplicated_gem"
      register_cop :DuplicatedGroup, "#{__dir__}/bundler/duplicated_group"
      register_cop :GemComment, "#{__dir__}/bundler/gem_comment"
      register_cop :GemFilename, "#{__dir__}/bundler/gem_filename"
      register_cop :GemVersion, "#{__dir__}/bundler/gem_version"
      register_cop :InsecureProtocolSource, "#{__dir__}/bundler/insecure_protocol_source"
      register_cop :OrderedGems, "#{__dir__}/bundler/ordered_gems"
    end
  end
end
{% endraw %}
{% endhighlight %}

That's it, your plugin is now ready for lazy-loading.

Three things to note:

- set minimum RuboCop version to `1.89.0` in the gemspec because of the new API
- add `extend LazyLoader` to the module so you can use `register_cop`
- provide an absolute path to the cop file in the second `register_cop` argument (hence the use of `__dir__`)

Depending on your plugin, there might be some other details to take into consideration. As a practical example, take a look at [this PR](https://github.com/rubocop/rubocop-rails/pull/1650) that added lazy-loading to `rubocop-rails`.

I encourage plugin authors to use the new API so we can all reap the benefits of faster RuboCop.

## Thanks Department

I got inspiration for this initiative when I saw [this issue](https://github.com/rubocop/rubocop/issues/14732) by [Jean Boussier](https://github.com/byroot) that included an idea to lazy load cops. While he started in the right direction, his proof of concept probably got stuck when it came to dealing with the cop registry and other bits. Nevertheless, it inspired me to give it a try. Thanks Jean!

After the first couple of PRs were merged, [Koic](https://github.com/koic) took over and [did gargantuan work](https://github.com/rubocop/rubocop/pull/15436) to finish everything. I'm super happy this happened because it sped up the most tedious part of the initiative. Thanks Koic!

Last but not least, thanks to RuboCop creator Bozhidar Batsov for being on board with the whole thing — it means a lot!

That's all, thanks for reading!
