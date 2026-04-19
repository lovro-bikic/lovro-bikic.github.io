---
layout: post
title: "rspec-mockbidden: Forbid Unwanted RSpec Mocks"
excerpt: "Utility gem for RSpec testing framework to forbid mocking methods on objects/classes/modules, evaluated at test runtime."
---

Proliferation of AI-generated code has caused a surge in mocks across test suites.

Some mocks I've seen have terrorized me:

{% highlight ruby %}
{% raw %}
let(:address) { Hash.new }

before do
  allow(address).to receive(:fetch).with(:country).and_return('HR')
  allow(address).to receive(:fetch).with(:province).and_return('Grad Zagreb')
  allow(address).to receive(:fetch).with(:postal_code).and_return('10000')
  allow(address).to receive(:fetch).with(:city).and_return('Zagreb')
  allow(address).to receive(:fetch).with(:street).and_return('Trg Republike Hrvatske')
  allow(address).to receive(:fetch).with(:house_number).and_return('15')
end
{% endraw %}
{% endhighlight %}

When I get this kind of slop on a pull request, I don't mind explaining there's an easier way to construct a hash:

{% highlight ruby %}
{% raw %}
let(:address) do
  {
    country: 'HR',
    province: 'Grad Zagreb',
    ...
  }
end
{% endraw %}
{% endhighlight %}

but, you know, repeating the same thing over and over again gets boring really fast.

[`rspec-mockbidden`](https://github.com/lovro-bikic/rspec-mockbidden) is a little gem that forbids such mocks for me:

{% highlight ruby %}
{% raw %}
RSpec.configure do |config|
  config.before do
    forbid_any_instance_of(Hash).from receiving(anything)
  end
end
{% endraw %}
{% endhighlight %}

Now every time someone tries mocking a method on a `Hash`, their test fails with an error:

`Hash is forbidden from mocking any instance method`

AI, however, is not the reason I created this gem; it's merely what pushed me to create it. The reason I created it is because, on codebases of a certain size, you need to communicate what is acceptable to mock and what should always execute.

For example, on the Rails codebase I'm working on, we use `FactoryBot` to set up test data with `create` and `build` methods. Code like this is not acceptable:

{% highlight ruby %}
{% raw %}
before do
  allow(User).to receive(:find).and_return(instance_double('User', first_name: 'Lovro'))
end
{% endraw %}
{% endhighlight %}

If we already have a database available in our test suite, we should use it (it will be used in production anyway, and that's the reality we want to simulate in tests). If we don't need database access in a test, `build` does the job.<sup>[^1]</sup>

`rspec-mockbidden` puts a stop to this:

{% highlight ruby %}
{% raw %}
before do
  forbid(ApplicationRecord).from receiving(:find)
end
{% endraw %}
{% endhighlight %}

Mocking `.find` is now forbidden on `ApplicationRecord` or any of its subclasses.

If needed, mocking all class methods on `ApplicationRecord` can be forbidden as well:

{% highlight ruby %}
{% raw %}
before do
  forbid(ApplicationRecord).from receiving(anything)
end
{% endraw %}
{% endhighlight %}

It also works the other way around, forbidding a specific method on all classes:

{% highlight ruby %}
{% raw %}
before do
  forbid(anything).from receiving(:create!)
end
{% endraw %}
{% endhighlight %}

For fun, you can also `forbid(anything).from receiving(anything)`, I won't stop you (though it's probably not realistic since mocks are sometimes valid).

I should note that when `.and_call_original` is used on a mock, an error won't be raised. This, for example, is perfectly acceptable:

{% highlight ruby %}
{% raw %}
before do
  forbid(ApplicationRecord).from receiving(:first)
end

it 'loads the first user only once' do
  allow(User).to receive(:first).and_call_original

  FirstUserService.call

  expect(User).to have_received(:first).once
end
{% endraw %}
{% endhighlight %}

`forbid` can be added anywhere in the test lifecycle:

{% highlight ruby %}
{% raw %}
it 'forbids in the test' do
  # valid only for the duration of this test
  forbid(User).from receiving(:first)
end

# valid for any test where this hook is applied
before do
  forbid(anything).from receiving(:call)
end

# mocking `ApplicationRecord#save` is forbidden for all tests in the suite
# this is probably a good place for suite-wide rules
before(:suite) do
  forbid_any_instance_of(ApplicationRecord).from receiving(:save)
end
{% endraw %}
{% endhighlight %}

[The README](https://github.com/lovro-bikic/rspec-mockbidden#rspecmockbidden) has more implementation details and examples, including installation instructions.

The gem has been released today as [v0.1.0](https://rubygems.org/gems/rspec-mockbidden/versions/0.1.0). It has been tested on two production codebases, but it's still an early version with a limited API. The API might be extended or changed, depending on real-world usage (e.g., forbidding multiple methods at once).

Feedback and contributions are welcome at [https://github.com/lovro-bikic/rspec-mockbidden](https://github.com/lovro-bikic/rspec-mockbidden)

I'm interested in hearing which mocks people want to forbid from their codebase.

Enjoy!

<br />

#### Footnotes

[^1]: [You might be interested in knowing how to ensure this really is the case.](/factory-bot-build-without-creating)
