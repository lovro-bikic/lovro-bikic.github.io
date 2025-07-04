---
layout: post
title: Making Sure FactoryBot Builds Without Creating
excerpt: "An RSpec spec to verify `FactoryBot.build` doesn't interact with the DB."
---

<a href="#recipe">Jump to recipe ↓</a>

---

[factory_bot](https://github.com/thoughtbot/factory_bot) is a great gem for writing factories in Ruby tests. It supports [multiple strategies](https://github.com/thoughtbot/factory_bot/blob/main/GETTING_STARTED.md#build-strategies) to instantiate records, the most familiar of which are `create` and `build`.

`build` simply returns an unpersisted record with set attributes and associations (also unpersisted, if configured properly), while `create` additionally saves it in the database.

Which strategy we use in which test depends on what the code does; if the code we're testing interacts with the DB, we probably want to create records. But if we're testing something DB-independent, like a model method that only uses attributes, then building is enough, and also preferable since it's much faster than creating.

Unfortunately, on most projects I've worked on, I've encountered (and, admittedly, sometimes written) factories that would execute SQL queries even when you'd build them, which leaves the test suite in an unoptimized state.

It would happen for various reasons, e.g. an association defined using `create`:

{% highlight ruby %}
{% raw %}
FactoryBot.define do
  factory :pet do
    veterinarian { create(:person) } # FactoryBot.build(:pet) still INSERTs a person in DB
  end
end
{% endraw %}
{% endhighlight %}

or a `build` callback that does something in the DB:

{% highlight ruby %}
{% raw %}
FactoryBot.define do
  factory :pet do
    after(:build) do |pet|
      pet.bite_the_vet! # presumably, updates last_bitten_at column on the poor vet
    end
  end
end
{% endraw %}
{% endhighlight %}

RuboCop can help somewhat here (e.g. [`FactoryBot/FactoryAssociationWithStrategy`](https://docs.rubocop.org/rubocop-factory_bot/cops_factorybot.html#factorybotfactoryassociationwithstrategy) cop will handle associations), but it can't cover all bases.

I recently spent some time optimizing the test suite on a project with more than 250 factories and plenty of traits. However, checking each factory’s build behavior individually was tedious. To help myself find non-buildable offenders — and prevent them from being introduced in the future — I wrote a spec that uses the handy [rspec-sqlimit](https://github.com/nepalez/rspec-sqlimit) gem to check that for me:

<div id="recipe"></div>

{% highlight ruby %}
{% raw %}
# spec/factory_bot_spec.rb
require 'rails_helper'

RSpec.describe FactoryBot do
  describe '.build' do
    described_class.factories.each do |factory|
      context "with factory :#{factory.name}" do
        it "doesn't execute SQL queries" do
          expect { build(factory.name) }.not_to exceed_query_limit(0)
        end

        factory.defined_traits.each do |trait|
          context "with trait :#{trait.name}" do
            it "doesn't execute SQL queries" do
              expect { build(factory.name, trait.name) }.not_to exceed_query_limit(0)
            end
          end
        end
      end
    end
  end
end
{% endraw %}
{% endhighlight %}

If any built factory executes an SQL query, the test will fail, and the output from `rspec-sqlimit` will show you which queries ran, helping you pinpoint the cause.

Of course, if some queries are unavoidable even when building (maybe some `SELECT`s), then you can pass additional options to the `exceed_query_limit` matcher to exclude them.

That's all, thanks for reading!
