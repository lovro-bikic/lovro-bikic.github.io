---
layout: post
title: Real-World Data RuboCop Offense Reports
excerpt: "How to run RuboCop on real-world data and generate offense reports to share with other people."
---

When contributing to RuboCop, I [often](https://github.com/rubocop/rubocop/pull/14448) [include](https://github.com/rubocop/rubocop-rails/pull/1501) [real-world](https://github.com/rubocop/rubocop-rspec/pull/2097) [offense](https://github.com/rubocop/rubocop/pull/14288) reports in pull request descriptions to show that cops I want to add or changes I plan to make will have a sizeable impact. I also do this because running anything on real-world data can surface potential implementation bugs.

In this post, I'll show where I gather real-world data from and how I generate reports that can be shared with others. The reports include GitHub/GitLab links to specific lines of code, and they look like this:

{% highlight ruby %}
{% raw %}
# https://github.com/gitlabhq/gitlabhq/blob/ba8e6fd9f408a6a6e0d20a5fe0d378aa63247065/app/models/concerns/mentionable.rb#L200
source.select { |key, val| mentionable.include?(key) }

# https://github.com/gitlabhq/gitlabhq/blob/ba8e6fd9f408a6a6e0d20a5fe0d378aa63247065/config/initializers/fog_google_list_objects_match_glob_support.rb#L34
**options.select { |k, _| allowed_opts.include? k }

# https://github.com/gitlabhq/gitlabhq/blob/ba8e6fd9f408a6a6e0d20a5fe0d378aa63247065/lib/gitlab/ci/reports/test_suite_comparer.rb#L31-L33
head_suite.failed.select do |key, _|
  base_suite.failed.include?(key)
end.values

# etc.
{% endraw %}
{% endhighlight %}

This is neat because you can see where the offenses have been caught (which, among other things, allows you to find false positives), they're syntax-highlighted, and anybody can open the link to view the offense in context. The offense message is not displayed because in such reports it is less important; the code matters more.

First, I'll talk about fetching real-world repositories on which to run RuboCop. Then, I'll explain how to generate the reports with a custom RuboCop formatter.

## Real-world repositories

When we talk about real-world Ruby data, it's important to distinguish the type of data we need. For contributions to `rubocop-rails`, we'd like to have Rails repositories; likewise for `rubocop-rspec`, `rubocop` itself, and other similar gems.

I most commonly contribute to those three gems, so having relevant repos comes in handy. Fortunately, kind folks have compiled repos with real-world applications that can be quickly cloned.

These are:
- [real-world-ruby-apps](https://github.com/jeromedalbert/real-world-ruby-apps) (for `rubocop`)
- [real-world-rails](https://github.com/eliotsykes/real-world-rails) (for `rubocop-rails`)
- [real-world-rspec](https://github.com/pirj/real-world-rspec) (for `rubocop-rspec`)

These repos use [submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) to fetch other public repos. After following the installation steps from one of these repos' READMEs, you'll have lots of Ruby code checked out locally. This is the code on which we'll run RuboCop.

The apps in these repos have their own `.rubocop.yml` files. While needed in the context of the project, we don't want project-specific configuration to affect our RuboCop runs, so we'll delete them:

{% highlight bash %}
{% raw %}
# cd to the real-world folder, then:
find . -iname '.rubocop.yml' \( -type l -o -type f \) -delete
{% endraw %}
{% endhighlight %}

This will delete both files and symlinks named `.rubocop.yml` recursively.

Now we have the code ready locally. Next step is to create the RuboCop formatter.

## Custom formatter

To print the offenses with links, I wrote a custom RuboCop formatter which you can find in [this GitHub gist](https://gist.github.com/lovro-bikic/d43d4eba38efe711d48f87a8575e5f8b). It's based on the Clang formatter, and it should replace the [clang file](https://github.com/rubocop/rubocop/blob/master/lib/rubocop/formatter/clang_style_formatter.rb) in your local clone of RuboCop (you can also replace any other formatter if you prefer that).

The formatter uses Git to get the current revision of the file and the remote where it's hosted, then constructs a URL from the rev and the line number, giving you a permalink. Also, unlike other formatters, if an offense spans multiple lines, it prints all lines instead of only the first one.

I keep the patch with the custom formatter [stashed](https://git-scm.com/docs/git-stash) locally, and whenever I need it, I just apply the patch.

## Running RuboCop

When I run RuboCop on real-world data, I typically run it against a single cop that I'm adding or modifying.

Assuming the real-world repo is cloned in the same parent folder as `rubocop`, you can generate the report by running RuboCop from the `rubocop` folder, for example:

{% highlight bash %}
{% raw %}
bundle exec rubocop ../real-world-ruby-apps/apps --only Style/HashSlice -f clang > report.rb
{% endraw %}
{% endhighlight %}

If all went well, `report.rb` should contain offenses for the cop (in this case, `Style/HashSlice`) for all applications from `real-world-ruby-apps`, formatted with public references. I save the report in a `.rb` file to get syntax highlighting in the IDE, which usually works, though highlighting may break depending on the offense code.

That's it. I wish you good fortune in the contributions to come.
