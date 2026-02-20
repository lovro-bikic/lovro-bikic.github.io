---
layout: post
title: "300 Days of RuboCop"
excerpt: "It's a story that begins with a pull request and ends with a Zen Buddhist saying."
---

> It all moves along, however crowded, quite steadily at the rate of 25 miles per hour (Terran). Gethenians could make their vehicles go faster, but they do not. If asked why not, they answer "Why?" Like asking Terrans why all our vehicles must go so fast; we answer "Why not?" No disputing tastes.

— Ursula K. Le Guin, The Left Hand of Darkness

<br />

Hi, my name is Lovro and I spent 300 days adding a linter to a legacy codebase with one million lines of code.

What follows is an account of my initiative to add RuboCop to a Ruby on Rails codebase. I'll talk about what set it off, my RuboCop proposal to management, why it got approved, my approach to enabling cops and fixing offenses, how I started contributing back to RuboCop, how others started contributing to the initiative, about fast and right solutions to problems, the opinions I gathered along the way, and, finally, when it was time to call it quits.

But I didn't know any of that when I started.

I just knew there was a pull request, and my review had just been requested.

## We've Been Here Before

I'm not shy when it comes to PR reviews. Incorrect code, performance optimizations, refactoring, simplification, stylistic nitpicks, poor grammar in documentation — all are equally deserving of a comment. When there's an opportunity for educating the author (common for juniors' PRs), I will jump on it. Sometimes, I also message the author with links to articles I think they could read.

One time, I left 230 comments on a PR. Nobody was happy about it, not me certainly, but I did tell the author many times before that big features need to be split into multiple PRs to make everyone's life easier. Despite my pleas for best practices, still I was requested to review 7k lines of code at once. Well, I haven't gotten a PR like that since.

Fast forward 5 years, I'm working on a different project in a big team. A massive legacy Rails codebase, code name Chaotic Beauty. Bugs cannot escape its gravitational field.

One day, a recently joined developer added a small-ish feature, and the PR link came my way. I opened it. I went through it once, scrolled to the top and started writing.

There were a number of issues with it. It was not unexpected; the dev was new to Ruby. I saw it as an educational opportunity for both parties. Out of the comments I left, I'll highlight two pertinent ones.

The first one was for a unit test:
{% highlight ruby %}
{% raw %}
# spec/lib/tasks/users/employees_backfill_education_degree.rb
RSpec.describe Tasks::Users::SetEmployeeEducationLevel do
  # tests go here
end
{% endraw %}
{% endhighlight %}

This test's unit lives in `lib/tasks/users/`**`set_employee_education_level`**`.rb`. Notice how the test path doesn't match the unit path. If I search for "set employee education level" in my editor, I'll find the unit, but not the test. The path is not intuitive. When this anti-pattern scales to thousands of files, good luck navigating the codebase.

The second one was for a test variable, which initially looked like this:
{% highlight ruby %}
{% raw %}
let (:user) {
  create(:user)
}
{% endraw %}
{% endhighlight %}

I commented that `create(:user)` can be on the same line as `let` (3 lines are not needed for simple record setup):
{% highlight ruby %}
{% raw %}
let(:user) { create(:user) }
{% endraw %}
{% endhighlight %}

He replied that the "syntax formatter is forcing this style". That was true, but it was forcing it only because the dev put a whitespace between `let` and `(:user)`, [which is not preferred](https://rubystyle.guide/#parens-no-spaces). It's a method call, after all. Once the whitespace was removed, the suggested one-liner was allowed by the formatter.

There were other similar comments. We racked up 82 in total ("a few", I think I said). I estimate my reviewing time was around an hour and a half. I guess it took the dev a bit more than that. And we both probably took a long coffee break afterwards.

In this estimation, I didn't even account for that feeling of lost time that lingers even when the coffee break is over and you need to move on with your day. The situation got me thinking: should I propose RuboCop, _again_?

<div style="text-align: center">* * *</div>

I had proposed RuboCop the first time about a year before. I suggested it to a staff engineer because we only had a syntax formatter at the time (and RuboCop is also a linter), but he was vehemently opposed to the idea. I can't recall his arguments.

Weeks after the exchange, the same SE figured out we needed to replace the syntax formatter, which was [Ruby plugin for Prettier](https://github.com/prettier/plugin-ruby). This time around, a colleague of mine mentioned RuboCop, but authority still wasn't swayed. Something about how he (SE) didn't have experience with it, and how he first wanted to turn on syntax formatting without linting (he wasn't aware RuboCop supports that), and that too much time was spent already selecting the new tool.

First we switched to [Rufo](https://github.com/ruby-formatter/rufo), then 3 weeks later we switched to [SyntaxTree](https://github.com/ruby-syntax-tree/syntax_tree). The switch required a code freeze and a massive commit to restyle the codebase GitHub would prefer you wouldn't open in the UI. It also caused collateral damage in the form of merge conflicts for the many open PRs, of which there were 150+ at the time (at any time, really).

Side note: you might like to know that Ruby plugin for Prettier is only a wrapper around SyntaxTree, meaning we ended up pretty much exactly where we began, with a little road bump in between. There is a guy pushing a rock up a hill who is very impressed by this.

I'm sure the ordeal didn't imprint itself as a happy memory on anybody, and I wouldn't have been thinking about switching formatters again were it not for the SE's departure from the company, some time after the ordeal.

I saw the window of opportunity. Other ears would listen now, what could I possibly lose?

## ROI, Nice To Meet You

I set up a channel (`#rubocop-in-monolith`), invited staff engineers and the engineering manager, then started writing. There was no plan, I let it structure itself naturally as I wrote. My first sentence was that I invited everyone because I wanted to open the discussion about enforcing RuboCop in the monolith.

The argument began by pointing out that we only have a syntax formatter, with which _"you can still make tons of mistakes and all it will do is format them nicely"_, as I wrote. As a matter of fact, it also adds trailing commas and forces single quotation marks for strings, but still, it's not a linter.

I continued by claiming we hastily switched formatters (linking to relevant discussions), and that we keep switching them because they're too simple and barely customizable (meaning, they have few rules, which can rarely be modified or disabled), something not acceptable for a monolith our size.

_"As another point, and I'm sure you're aware of this, there's a fair amount of non-Ruby devs working on the monolith. Syntax formatting is not enough for them. It's not enough even for me, and I've been writing Ruby almost exclusively for the past 5+ years."_ I linked to the 82-comments PR, pointed out that only a minority of comments touched upon the business logic (yes, I'm partly to blame here), and also mentioned the amount of time it took to review it.

Why RuboCop? Because:
- it's a syntax formatter _and_ linter, with [hundreds of available rules](https://github.com/rubocop/rubocop/blob/master/config/default.yml) (known as cops),
- there are plugins with additional cops for different domains (plugin for [Rails cops](https://github.com/rubocop/rubocop-rails/), for [RSpec cops](https://github.com/rubocop/rubocop-rspec), for [FactoryBot cops](https://github.com/rubocop/rubocop-factory_bot/), etc.),
- it's a [de-facto standard](https://www.ruby-toolbox.com/categories/code_metrics) in the Ruby community for static code analysis,
- it's used by big players like [Shopify](https://github.com/Shopify/ruby-style-guide), [GitHub](https://github.com/github/rubocop-github), [Airbnb](https://github.com/airbnb/ruby), [Discourse](https://github.com/discourse/rubocop-discourse), [The Government of UK](https://github.com/alphagov/rubocop-govuk), and so on,
- it's used for Ruby on Rails itself,
- we can write custom cops to enforce rules specific to our domain,
- it's highly configurable (each cop can be disabled, and cops typically have configuration options),
- it can be introduced gradually, cop-by-cop, with minimum friction (no code freezes and minimum merge conflicts),
- it can co-exist with the current formatter until RuboCop is ready to take over.

I should have (but didn't at the time) linked to specific cops like [`RSpec/SpecFilePathFormat`](https://docs.rubocop.org/rubocop-rspec/cops_rspec.html#rspecspecfilepathformat) and [`Lint/ParenthesesAsGroupedExpression`](https://docs.rubocop.org/rubocop/cops_lint.html#lintparenthesesasgroupedexpression) that would have actually made my comments for unit test path and the test variable in that PR obsolete, thereby directly showing how RuboCop can save time.

I looked over what I wrote a couple of times, then clicked Send.

<div style="text-align: center">* * *</div>

There is a presence in most stakeholder meetings. It takes many forms. Newcomers might be oblivious to it; others understand it instinctively. It manifests itself as employee productivity, time savings, customer retention, cost optimization... just to name a few. It is the life and death of technical initiatives. It is the invisible stakeholder with power of veto. It has a name too. We call it Return on Investment.

The technical crowd in the channel provided valid counterpoints for RuboCop. Its high configurability is a double edged sword because it can introduce bikeshedding, and requires strong ownership. False positives are not uncommon. It can pollute Git history ([ignore revs file](https://git-scm.com/docs/git-blame#Documentation/git-blame.txt---ignore-revs-filefile) will help here). Some cop departments are more useful than others. We debated, more out of pure technical interest than anything else.

But out of everything I wrote, one point struck a major nerve: why are developers spending their time on things on PRs that can be resolved with tools, and not focusing on business logic? I was asked to estimate the effort of adding RuboCop.

The discussion continued for a while longer.

Stakeholders said they'll get back with a decision. Two weeks later, it was approved. The scope was defined, I was designated the owner of the initiative, and work could begin. The feeling was a mixture of happy and anxious. I'm not shortsighted, this thing will obviously take a lot of effort. I have no idea what's waiting for me.

Would I have succeeded had the staff engineer still been here? For one, it would have been rude to go behind his back and propose RuboCop a third time within a year, and this time to his leads. Alas. There's a [paraphrase of Max Planck](https://en.wikipedia.org/wiki/Planck%27s_principle) that science progresses one funeral at a time; I guess a company's unit of progress is a senior departure.<sup>[^1]</sup>

## This Mess We're In

I added RuboCop and set up the CI workflow. The initial configuration looked like this:

{% highlight yaml %}
{% raw %}
# .rubocop.yml
AllCops:
  NewCops: enable
  DisabledByDefault: true
{% endraw %}
{% endhighlight %}

The plan was to work on [Lint department](https://docs.rubocop.org/rubocop/cops_lint.html) first, arguably the set of cops with the most useful rules. The claim will be substantiated shortly.

I opted to fix offenses not by auto-generating a [`.rubocop_todo.yml`](https://docs.rubocop.org/rubocop/configuration.html#automatically-generated-configuration) file, but by enabling or disabling one cop at a time from an empty config. There were no plans to mass fix multiple cops in huge PRs, even cops with safe autocorrect.

You can generate a report with the count of offenses per cop by running `rubocop --format offenses` (just make sure to comment out `DisabledByDefault`). The file is sorted by count, max offenses at the top. I worked from the bottom. Starting with cops that have only 1 or 2 offenses prevents one from being overwhelmed; that the first report had offenses for 450+ cops was overwhelming enough.

With the report in hand, I started fixing. Come with me on a road trip through my first PRs:

{% highlight ruby %}
{% raw %}
# offense for Lint/BinaryOperatorWithIdenticalOperands
if address.lng.nil? || address.lng.nil?
  ...
end

# the author probably wanted to check if `address.lat` is nil too
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/EmptyWhen
case foo
when BAR,
     do_this_please
end

# the comma should not be there, `do_this_please` was intended to be executed when `foo` matched `BAR`
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/DuplicateRescueException
begin
  something
rescue StandardError
  handle_error
rescue StandardError
  handle_error
end

# probably a result of a poorly handled merge conflict
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/InterpolationCheck
context 'when state is #{state}' do
  ...
end

# false positive if state really is '#{state}'
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/RedundantSafeNavigation
SomeModule&.foo?

# why did someone think `SomeModule` could be nil?
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/Void
def items
  [] if DECLINED_STATUSES.include?(status)

  super
end

# the author intended the first line to be a guard clause, i.e. `return []`
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/UnreachableCode
def filter_discounts(discounts)
  return discounts

  discounts.select(&:applicable?)
end

# a quick business decision determined all discounts are applicable
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/DuplicateMethods
class Foo
  def bar
    does_something
  end

  # ...200 lines later...

  def bar
    does_something_else
  end
end

# once again, probably a poorly handled merge conflict
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/SafeNavigationChain
user&.address.country

# if `user` is nil, `.country` will raise `NoMethodError`
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# [insert offense for Lint/UselessAssignment]
# so many variables assigned but never used... it took 9 PRs to remove them all
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/ConstantDefinitionInBlock
RSpec.describe Foo do
  MAX_RUNS = 4
end

# while it might not look like it, `MAX_RUNS` is in fact a global constant
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/ConstantReassignment
class Foo
  BAR = :bar
  # couple of lines later
  BAR = :bar
end
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
{% raw %}
# Lint/SuppressedException
def fetch_first_key(hash, *keys)
  keys.each do |key|
    begin
      return hash.fetch(key)
    rescue KeyError
    end
  end
end

# exceptions used for control flow, a classic
{% endraw %}
{% endhighlight %}

Turns out, we needed a linter after all.

As I was fixing offenses, I also publicly documented PRs which fixed bugs. One has to feed ROI.

The work quickly turned into a routine. Turn a cop on, fix offenses, then pick another cop. Like an old man doing crosswords. And I would more than happily have continued emptying the report like this, had I not suddenly come to a halt.

## The First Fork in the Road

One day, I enabled [`Lint/FloatComparison`](https://docs.rubocop.org/rubocop/cops_lint.html#lintfloatcomparison). It warns you when you use floats in (in)equality comparison because it's unreliable (due to precision loss). But then it reported an offense for this line of code:

{% highlight ruby %}
{% raw %}
Float(latitude, exception: false) == nil
{% endraw %}
{% endhighlight %}

That didn't seem right, it only checks if variable `latitude` is an invalid float string. I didn't use [`Kernel#Float` method](https://docs.ruby-lang.org/en/4.0/Kernel.html#method-i-Float) before, but I ran it in the console and sure enough, given a string that cannot be parsed as a float, the left hand side of the expression returns `nil`. This code is correct.

Unknown to me at the time, this was a fork in the road.

The easy solution was this:

{% highlight ruby %}
{% raw %}
Float(latitude, exception: false).nil?
{% endraw %}
{% endhighlight %}

Refactor and move on. It's even idiomatic Ruby. But that didn't sit right with me.

It didn't sit right with me because there might be more similar offenses in existing code (there were) or some dev might add similar code in the future and then be confused by the offense (wasting billed time, which does not appease ROI). But ROI aside, here's the bigger picture: everyone who uses RuboCop will have devs likewise confused by the offense. Some might even think they're doing something wrong.

I stared at the offense, and it stared back at me.

It was really hard _not_ to look into the RuboCop source code — which I've never opened before — now being motivated by curiosity, economics, and a budding sense of accomplishment.

[So I looked](https://github.com/rubocop/rubocop/pull/13432). I even said Hi! on my first PR, I think it's good manners.

The first time is always the hardest. After searching GitHub issues and not finding mine, I forked the repo. RuboCop uses RSpec for its test suite, so that was familiar to me. I copied one of the existing tests for the cop and modified it to fit my scenario. It failed, as expected.

Then the hard part. Suffice it to say, after one too many breakpoints, I finally figured out the solution. Happy with it, I ran all checks and they passed. I don't know why, but I was so nervous to open the PR that I re-read my patch at least ten times! Grammar, logic, formatting, PR checklist — I didn't want to mess anything up. And it was only a few lines of code. When I finally realized I was being ridiculous, I clicked Open.

It was merged 65 minutes later. And I got a "Thank you!" What a dopamine hit. Followed by another, a few weeks later, when the PR was mentioned in [v1.69.0 release notes](https://github.com/rubocop/rubocop/releases/tag/v1.69.0).

After the first hit, I got interested. And with hundreds of cops still left to enable on a gargantuan project, it was the perfect environment to start an avalanche of contributions. But I will cover the lessons learned from the [70+ contributions](https://github.com/search?q=org%3Arubocop%20author%3Alovro-bikic&type=pullrequests) that came from this and still keep coming in some other post. Back to the crossword. But first, a short break.

#### AI Pit Stop

I asked one of the popular LLMs about the offense, and it told me to use `Float().nil?` (easy solution) or to disable the cop (ignore-the-problem solution).

I continued prompting for different ideas — taking care to not be explicit as to the desired approach — but it kept proposing more code solutions, each more complex than the last (e.g., to save result of `Float()` in a variable, then check if it equals `nil` on a separate line, thereby avoiding the cop). Only when I pointed out that `== nil` is correct, it conceded the offense is a false positive and the best course of action is to report the issue. I guided it there.

But what if you let AI guide? If you didn't know any better, you might have stopped at `Float().nil?`. AI tools certainly make that easy,  when such solutions are only a keyboard shortcut away (and they come about even faster with agents). But amidst all that productivity, where did the time to stop and think go?

A fork in the road is a time to stop and think. Mine was a trivial one, yet it led me a long way. It could have just as easily led me nowhere. In fact, most forks _do_ lead nowhere, but that just makes them more exciting when they don't. But I never could have taken it had I not slowed down enough to see it, and not just be led the other way.

## Applying the Boy Scout Rule

Ticking off cops from the report, I came upon [`Lint/UnusedMethodArgument`](https://docs.rubocop.org/rubocop/cops_lint.html#lintunusedmethodargument). Here's an illustrative offense:

{% highlight ruby %}
{% raw %}
def payment_settings(country, user)
  Payments::Client.settings(country)
end
{% endraw %}
{% endhighlight %}

There were 1.1k offenses for the cop. RuboCop's autocorrect will help you resolve all offenses quickly, by simply doing this:

{% highlight ruby %}
{% raw %}
def payment_settings(country, _user)
  Payments::Client.settings(country)
end
{% endraw %}
{% endhighlight %}

The number of parameters is still the same, so existing method calls won't break, and `_user` is clearly marked as unused. The solution is fast and correct. But is it the right one?

One of the goals of the initiative was to remove everything that's redundant. Redundant code increases mental overhead. `_user` signals that we don't have to pay attention to it, but it's still there, and we have to provide it when calling the method. Plus, it got there because the method _used to_ use the `user` argument, but then it was refactored to use only `country`. Meaning, this argument is a relic of the past. And past belongs in Git history.

So, I refactored all method callers to pass only `country` (and refactoring _their_ callers as well if they didn't need `user` anymore), then finally cleaned up the signature:

{% highlight ruby %}
{% raw %}
def payment_settings(country)
  Payments::Client.settings(country)
end
{% endraw %}
{% endhighlight %}

That looks right now, no?

This might not sound supportive, but I don't see the `_user` solution as adding much benefit to an already legacy codebase. If anything, it's only a bit less noise. If you want to commit to your own RuboCop initiative, ask yourself whether you just want to get the job done or whether you want to leave the project in a better state than the one it's currently in.

## Moving Along Steadily

In Le Guin's [The Left Hand of Darkness](https://en.wikipedia.org/wiki/The_Left_Hand_of_Darkness), there is a chapter about the perilous journey Genly Ai (who is a human) takes over the Gobrin ice sheet over 80 days. Proportionally speaking, it is much much longer than the novel's other chapters. It is a gruelling trip, but also an important one and rather beautiful. If it were shorter, you wouldn't feel it.

Don't worry, I'm not mentally preparing you for a long section. While it took 300 days<sup>[^3]</sup>, the bulk of it was repetitive enough to not be novel material.

Part of the reason it took that long is because I didn't spend my whole days on it. I wouldn't do that even if I could; I'd go mad just working on linter offenses all day long. I squeezed this work in between regular work that brought money, which turned out to be anywhere between 0 and 3 hours a day. Plus, other free time was invested in RuboCop contributions.

The only rule of thumb I adhered to was that PRs needed to be small enough to be reviewable. Grouping multiple cops in a single PR was okay as long as there were one or two offenses per cop.

One thing I learned is how to say No to scope creep. Sometimes, when fixing a cop's offenses, opportunities to refactor methods and classes presented themselves. They seemed to go naturally with the fixes. Despite their allure, it is imperative **not** to do those refactors in the same PRs, no matter how small they are. Justifying them with "Just this once, it's a small change" is a mental trap. Make a note and fix them later.<sup>[^2]</sup> Otherwise, you'll never get work done.

As a point of reference, I've enabled or fixed offenses for ~450 cops over the course of 360 PRs. If that sounds like a lot of work, that's because it is, and not without reason.

RuboCop has two modes of autocorrection: safe and unsafe. Safe autocorrect ["indicates whether the autocorrect a cop does is safe (equivalent) by design"](https://docs.rubocop.org/rubocop/usage/autocorrect.html#safe-autocorrect). In theory, you could fix offenses for all safe cops in one PR and be done with it because the code will still produce equivalent results. In theory.

When I enabled [`Style/SoleNestedConditional`](https://docs.rubocop.org/rubocop/cops_style.html#stylesolenestedconditional), which has safe autocorrect, it corrected this:

{% highlight ruby %}
{% raw %}
unless foo && bar
  5 if baz
end
{% endraw %}
{% endhighlight %}

to this:

{% highlight ruby %}
{% raw %}
if !foo && !bar && baz
  5
end
{% endraw %}
{% endhighlight %}

Fans of De Morgan's laws will notice something amiss. `unless foo && bar` is equivalent to `if !(foo && bar)` (or `if !foo || !bar`), not `if !foo && !bar`. But if you're not careful enough, or trust the idea of safe autocorrection too much, this [3-year-old bug](https://github.com/rubocop/rubocop/pull/14012) might evade you, even during a PR review.

RuboCop is a good tool, but it's still made by humans. That's why RuboCop's changes should be treated just like anybody else's. You would review a human's changes, so you should review RuboCop's too. Reviews usually aren't effective when PRs are big. Hence, one small PR at a time. There are no shortcuts here.

Well, not without a cost anyway...

## A Cowboy Takes the Shortcut

A test suite with high code coverage can help with some cops.

[`Style/MutableConstant`](https://docs.rubocop.org/rubocop/cops_style.html#stylemutableconstant) is a terrific cop, but it comes with a caveat: when it freezes a constant, all code paths which try to mutate it will raise an error:

{% highlight ruby %}
{% raw %}
class Shop
  DEFAULT_ITEMS = [:bicycle, :camera].freeze

  def self.sellable_items(user)
    items = DEFAULT_ITEMS

    # raises `FrozenError` because `<<` mutates arrays
    items << :fridge if user.fridge_supported?

    items
  end
end
{% endraw %}
{% endhighlight %}

When `.freeze` is added to `DEFAULT_ITEMS` array, the only way you can be sure this code will still work is if `Shop#sellable_items` has sufficient code coverage.

The situation I found myself in when I wanted to enable the cop was this: code coverage was low (< 20%), and there were more than 5200 constants that needed freezing. It's not an ideal situation.

When I saw the offense report, I couldn't help but be overwhelmed. Unlike other cops with many offenses (even cops without autocorrection support), the "right" way to fix offenses was tedious: go constant-by-constant and check each reference to see if it's mutated at some point (which requires reading all relevant code and knowing which APIs have side effects).

Assuming a constant is referenced an average of 2 times (most constants are referenced only once, but some are referenced 100+ times), and that each reference takes 10 seconds to read through (optimistic, yes), 10400 references would take at least ~29 hours to check. Even my tenacity has its limits.

I debated with myself how to resolve the challenge, but didn't have a good solution in mind. When I tried freezing all constants in a dry-run PR, the CI failed with errors for the code that _was_ covered by tests, but I assumed that was just the tip of the iceberg. And because of our staging environment setup, deploying the dry-run PR for QA testing would be feasible only for critical flows, which were already covered by unit tests.

Out of ideas, I decided to autocorrect the offenses and deploy to production, accepting the possibility of errors. I told myself that code which mutates constants is buggy anyway, so temporary errors aren't really that much worse than "working" bugs. I announced in the initiative channel what I was about to do and what it entailed, and also that the work will be spread over a couple of days to soften the blow. We also have application monitoring, I could keep my eyes peeled for `FrozenError`.

I merged one PR a day over the course of a week. As expected, there were errors. Three, to be exact, or 0.06% of all frozen constants. And fortunately all of them in non-critical code paths. That's the thing about icebergs, you never know how deep they run.

This was the only cop in the initiative I handled the cowboy way. I didn't know any better then. I don't recommend the approach because it's not professional, but sometimes it's a cost-effective solution. If you'll take that approach, the least you can do is be upfront about it and to spread the impact.

#### AI Pit Stop 2

As I was finishing up this section of the post, it occurred to me that I could have asked AI to give me ideas for the problem. This might sound inconsistent with my opinion on AI in the first pit stop, but it's actually not.

I'm not dismissive of AI; it's just a tool. A tool has its uses. Each use has its advantages and disadvantages. It takes time to be aware of them. Taking time is not being dismissive, but it might certainly look like it in this ever accelerating world.

Anyway, I wondered whether I could track constant mutations somehow, so I asked one of the reasoning models to give me a couple of ideas. The options it provided gave me an idea to use [`Module#const_added`](https://docs.ruby-lang.org/en/4.0/Module.html#method-i-const_added) and a custom Rack middleware to figure out _which_ constants are mutated in a request by comparing the state before and after, which would be enough information for me to then track down the mutation in code. In the end I had a working solution which I could use in production. It comes with some performance and memory overhead, but for temporary instrumentation I think that's acceptable.

Looking back, I wish I had spent a bit more time thinking before freezing all constants. I looked into official documentation, Stack Overflow questions, various forums and read through Ruby issue tracker, but still, I didn't connect the dots. In such cases, I don't think using AI is a bad idea to explore options one might have missed or hasn't thought of.

## Sharing the Sheriff Badge

One of the advantages of RuboCop I highlighted is the ability to write custom cops. This would allow us to enforce rules specific to our codebase, for many people who come and go on the project. I knew that some big players utilize this feature well (GitLab, for example, [has over 200 custom cops](https://gitlab.com/gitlab-org/gitlab/-/tree/389db2cb5e7a33720d1f8333ea7d902f2d3a87c2/rubocop/cop) at the moment), but I didn't have any ideas in mind for our monolith, even when I was already a couple of months into the initiative.

Turns out, other people would come up with ideas for cops themselves.

We use the [Typhoeus](https://github.com/typhoeus/typhoeus) gem for HTTP requests to different external services. By default, the gem doesn't enforce a timeout for the total request duration. You could set a global timeout as a fallback, but that won't cut it when some services should allow max 5 seconds, and others even up to 1 minute. The problem we had was that many places in our codebase didn't use a timeout, which was a big issue for app reliability.

So, one dev came up with a cop to scan usages of Typhoeus API and enforce setting a timeout. With it, we identified all places where timeouts were missing, fixed them, and also made sure all future devs will be made aware you don't want to run HTTP requests forever.

A month later, a cop for DB migrations appeared. Then, one for freezing YAML objects on load, another for not setting an explicit autoincrement column in factories, and so on. As patterns and anti-patterns emerge on the project, I'm sure even more cops will be created in the future. And if they're generic enough, we can even try upstreaming them to RuboCop.

Of course, not every pattern should be a cop. Custom cops make sense when you can reliably implement them (with little probability of false positives) and when they would catch offenses often enough (spending 3 hours writing a cop to catch 2 nitpick offenses in a year is probably best left to manual review). A cop is something you have to write, test, document and maintain, so the cost must be justified.

And when cost is involved, there's one cardinal mistake you don't want to make.

## Staying on ROI's Good Side

The days were going by. Cop after cop, department after department, the report was getting shorter and shorter. After Lint, I turned to FactoryBot, RSpec, Security, Naming, Rails, Performance, Style and Layout departments. I worked sometimes by department and sometimes by the number of offenses.

FactoryBot, I enabled almost all cops. RSpec, Security and Naming too. We needed about a half of Rails cops. A third of Performance cops, the ones which didn't impact readability that much. A little more than a half of Style cops. The whole Layout department (with some customizations).

I disabled the [Metrics department](https://docs.rubocop.org/rubocop/cops_metrics.html) altogether. Existing classes and methods were long and complex, and refactoring all that code was not in the initiative's scope. But even if this were a new project, my experience with this department is that the rules are just too context-dependent to be generally useful.

To find out how others feel about Metrics, I did a little analysis of [real-world-rails](https://github.com/eliotsykes/real-world-rails) for `rubocop:disable` directives. Here are statistics for disabled RuboCop-only cops grouped by department, at the time of writing:<sup>[^4]</sup>

{% highlight text %}
{% raw %}
 951 Metrics/
 503 Style/
 419 Lint/
 415 Layout/
 300 Security/
 206 Naming/
   5 Gemspec/
   4 Bundler/
{% endraw %}
{% endhighlight %}

Metrics cops are disabled almost twice as much as the next most disabled department. Out of the top 10 disabled cops, 6 come from Metrics:<sup>[^5]</sup>

{% highlight text %}
 359 Layout/LineLength
 328 Metrics/AbcSize                  1
 243 Security/PublicSend
 201 Metrics/MethodLength             2
  93 Metrics/CyclomaticComplexity     3
  87 Metrics/BlockLength              4
  78 Metrics/PerceivedComplexity      5
  68 Metrics/ClassLength              6
  62 Lint/InterpolationCheck
  49 Naming/PredicateName
{% endhighlight %}

I don't think Metrics cops are without merit (sometimes an offense will prompt me to refactor a method and I'll end up with something cleaner), but the noise of directive comments is also something to consider. Plus, with rampant use of AI, the models tend to bend code over backwards to comply with the cops. Food for thought.

When there were around 50 cops left in the report, I started feeling saturated. It wasn't that I got bored of the initiative, it was more that the remaining cops seemed to become [less and less useful](https://en.wikipedia.org/wiki/Diminishing_returns).

It began with a cop like [`Style/DoubleNegation`](https://docs.rubocop.org/rubocop/cops_style.html#styledoublenegation). Like yes, I could change `!!something` into `!something.nil?`, sure. Or I could break up couple of chained multiline blocks to tick off [`Style/MultilineBlockChain`](https://docs.rubocop.org/rubocop/cops_style.html#stylemultilineblockchain) from the report, but why? Like Metrics, I don't think these cops are without merit, but after I've fixed bugs, removed redundant code, simplified things, improved consistency, performance and syntax formatting, these remaining cops felt like playing with variations of the same code, one not that much better than the other.

So when I looked at the report and saw only such cops, I knew my job was done.

That is, from the technical side at least.

## Don't Forget About Others

Being intimate with RuboCop, it's easy for me to overestimate how much the average developer is at ease with it, and easy to forget I'm not the only one using it. _Sigh._ The fun part is over, documentation drudgery begins.

I opted to write docs in the repository README. They explain:
- what is RuboCop and what we are using it for (linting and syntax formatting),
- terminology (rule == cop, rule violation == offense, cop category == department),
- how to run RuboCop locally,
- how to autocorrect only layout offenses (`-x` flag),
- the difference between safe and unsafe autocorrection,
- our policy for disabling cops (which is: prefer not to, but if you must, explain the decision in the cop disable comment),
- editor integrations (e.g. Ruby LSP, Solargraph), and
- how to deal with CI failures.

If you're using RuboCop daily, these items might seem obvious and not worth documenting. Just remember that there was a point in time when you didn't find them obvious either.

GitLab is also a [good example](https://docs.gitlab.com/development/rubocop_development_guide/) in this regard. But whatever you choose to write, just make sure to adjust the language to the skill level of your team.

Burning issues are also a part of the initiative. Sometimes, CI would fail for random reasons. Other times, new code with offenses would be merged to `master` minutes after I enabled a cop. Less burning, but sometimes people would ask how to fix offenses for some cops and we'd discuss it. And then there was scheduled work, like periodically updating RuboCop and fixing new offenses.

On the topic of failing CI, I think it's a good idea to check other people's failing runs to see how adoption is progressing, and if there are areas for improvement. I typically spot check the runs from time to time. Also, since our CI has an API for fetching workflow logs, I wrote a script to extract some offense statistics.

Here are, for example, the top 20 failing cops for the past ~3k workflow runs:

{% highlight text %}
 486 Layout/MultilineMethodCallIndentation
 451 Layout/LineLength
 364 Style/FrozenStringLiteralComment
 297 Layout/TrailingWhitespace
 227 Layout/TrailingEmptyLines
 148 Style/TrailingCommaInArguments
 146 Style/RedundantConstantBase
  74 RSpec/VerifiedDoubles
  57 Style/TrailingCommaInHashLiteral
  56 Lint/Syntax
  53 Style/IfUnlessModifier
  48 Rails/Output
  48 Layout/FirstHashElementIndentation
  43 Layout/IndentationConsistency
  37 Rails/Blank
{% endhighlight %}

There were 3432 offenses in total for 151 cops, meaning the top 7 cops (~5%) account for ~62% of offenses. Notice how the top 7 cops are autocorrectable (one exception being `Layout/LineLength`, but only sometimes).

This suggests I could raise tooling awareness in the team. For example, I could tell people that some editors support trimming trailing whitespace on save (thereby avoiding `Layout/TrailingWhitespace` offenses). Or, I could come up with a solution to autocorrect some of these cops on file save, which people could then integrate with their editors (these cops' safe autocorrect can generally be trusted).

One more use of this data is to check how many would-be issues and bugs have been prevented without PR reviews. While not shown in the top 20, RuboCop also caught Lint offenses similar to those when I just started the initiative. And while I'm sure most would be caught during a PR review, this is still saved time for everybody.

Speaking of time. I believe I've shared every relevant fact and figure, and probably one too many opinion. But now, almost 6 months after the final PR, I don't think much about any of that. What I do think about, though, is one question:

## Twenty Seven Thousand Green Dots

Given the opportunity, would I do it all again? That is, if I were to work on another huge legacy codebase, would I go through the same trouble of convincing management we need a linter and setting off months of menial work?

Yes, with a caveat.

When I began, I didn't know that much. I didn't know that what makes management accept an initiative is not a well-reasoned essay, but usually some guy called ROI who doesn't care whether you like him or not. And I didn't know how to start working, except by fixing one offense, and then another. I also didn't know how to contribute to one of the most popular Ruby libraries out there, which, I previously assumed, was a done project that would have little use of me. And I for sure didn't know how much time it would all take, though I did give an estimate (for some reason, I always have to).

But what I truly didn't know is where that time would take _me_, and what I now know is where it could take somebody else.

My 300 days taught me many things. Another 300 would probably teach me a bit more (though honestly, I'd be on the lookout for RuboCop contributions the most), but I also know that diminishing returns are a thing, and that, even if they weren't, interests change. Mine certainly have.

Therefore, my caveat is that I'd only do the whole thing again, but as a mentor.

There is, in fact, a sea of opportunity in menial, repetitive work; we just don't tend to see it. The reason is simple.

We underestimate such work from the very start by assuming there's nothing to learn from it. Because of that assumption, we don't focus on the work fully. By not focusing fully, we miss the hidden opportunities. By missing the hidden opportunities, we end up exactly where we assumed we would.

<div style="text-align: center">* * *</div>

I've given glimpses into the monolith with figures and examples. One figure I didn't give is that the 1+ million lines of code are split between 27 thousand files. There's a quote [attributed to Bertrand Russell](https://quoteinvestigator.com/2013/02/20/moved-by-stats/) that _"The mark of a civilized man is the capacity to read a column of numbers and weep."_ I'm not sure he had monoliths in mind.

As much as I'd like to, I can't weep over a million lines of code. It's a number that sounds great, but my senses just don't react to it. A couple thousand, on the other hand, is a magnitude much familiar to my senses. After all, I've been looking at it for 300 days.

The chaos that magnitude brings is out of control of any living being. Good engineering can create lasting architecture, optimize API endpoints, reduce infrastructure costs, succinctly and transparently document decisions, write code that can change almost as fast as business decisions, but it can't keep track of all lines of code forever. Tools can.

<div style="text-align: center">* * *</div>

When you run RuboCop, by default it will print a character per checked file to terminal output. An offense might result in a red <span style="color: red">E</span> (error) or a magenta <span style="color: magenta">W</span> (warning). For no offenses, you get a green dot.

I must have run RuboCop hundreds of times during the initiative. The slowness on a huge codebase bothered me enough [to optimize it for CI](/github-actions-rubocop-workflow/), which I then [upstreamed to Rails](https://github.com/rails/rails/pull/54754). Each run would first pause for a couple of seconds as RuboCop was warming up, then dutifully print out 27 thousand characters to inform me of the verdict. It was so annoying to see one <span style="color: red">E</span> in what was otherwise a sea of green dots.

But overall, it was good work and I liked it. I like what it brought, and that I didn't know what that would be.

<br />

<span style="color: green; overflow-wrap: break-word;">
........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................ Chop wood, carry water, fix offense
</span>

<br />
<br />
<br />
<br />
<br />

#### Footnotes

[^1]: If you find the statement cold, just know I'm a senior too.

[^2]: `Lint/UnusedMethodArgument` is an exception since the autocorrect would produce unfavorable code.

[^3]: 301 days to be precise, but developers are permitted off-by-one errors for marketing purposes. The first PR was on Oct 30, 2024, the last one on Aug 26, 2025. There were some PRs after that, but they were for periodic RuboCop updates as part of post-initiative support.

[^4]:
    This is the command I ran in the terminal on the `real-world-rails` repo to get the disable directive statistics:<br/>
    `grep -ohrE "# rubocop:disable .+" real-world-rails/apps | grep -oE "(Bundler|Gemspec|Layout|Lint|Metrics|Naming|Security|Style)/" | sort | uniq -c | sort -nr`

[^5]: The command is very similar to above command, and is left as an exercise for the reader.
