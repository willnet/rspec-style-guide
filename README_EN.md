# RSpec Style Guide
## What's this?

Etiquette for writing test code with high readability

We would like to improve our style guide by hearing all of your opinions, so please open issues and pull requests that correspond to the following:

- Doubts or questions
- RSpec etiquette that isn't written here
- The content is fine but the way it's expressed is strange
- Better sample code

# Presumed knowledge

- RSpec
- FactoryBot

## `describe` and `context`

`describe` and `context` are the same methods, but they can be used in the following ways to help differentiate the type of code you're testing.

- The argument of `describe` is what is being tested
- The argument of `context` is the presumed condition/state of the test when it is run

# Example

```ruby
describe Stack do
  let!(:stack) { Stack.new }

  describe '#push' do
    context 'when push is called on a string' do
      before { stack.push('value') }

      it 'verifies the returned value is the same as the pushed value' do
        expect(stack).to eq 'value'
      end
    end

    context 'when nil is pushed' do
      it 'raises an ArgumentError' do
        expect { stack.push(nil) }.to raise_error(ArgumentError)
      end
    end
  end

  describe '#pop' do
    context 'when the stack is empty' do
      it 'returns nil' do
        expect(stack.pop).to be_nil
      end
    end

    context 'when the stack has values' do
      before do
        stack.push 'value1'
        stack.push 'value2'
      end

      it 'retrieves the last value' do
        expect(stack.pop).to eq 'value2'
      end
    end
  end
end
```

# Default values in FactoryBot

When using FactoryBot, you can create default values for the columns of each of your models. It's good to create random variables for each column when you do this. Also, by explicitly defining only which values are necessary inside the tests, it makes it easier to understand which values are the most important.

### A bad example

For example, we have a column named `active` which will determine whether a user account is active or not. You will often see cases like this where the active/inactive column is fixed.

```ruby
FactoryBot.define do
  factory :user do
    name { 'willnet' }
    active { true }
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user, name: 'maeshima' }
    let!(:receiver) { create :user, name: 'kamiya' }

    it 'sends a message correctly' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

In this test, `User#active` is implicitly set to `true`. When values like `sender.active #=> false` and `receiver.active #=> false` are not explicit, you can't tell what effect these conditions will have.

Also, `name` is explicitly defined, but is it really needed? It's usually best to stay away from unneeded data that the reader will be looking at.

### A good example

```ruby
FactoryBot.define do
  factory :user do
    sequence(:name) { |i| "test#{i}" }
    active { [true, false].sample }
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user }
    let!(:receiver) { create :user }

    it 'sends a message correctly' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

With this test, you can tell (although it is implicit) that the return value of `User#active` does not have an effect on the behavior of `User#send_message`. If any changes in the future have an effect on `User#active`, you should be able to tell that the test has failed when the CI tests don't pass every now and then.

## Besides `belongs_to`, don't create default values for associations with FactoryBot

When making a model with FactoryBot, you can make models that are associated with it at the same time.

There isn't really a problem when using `belongs_to`, but you need to watch out when using `has_many`

As an example, we'll define a User that has many posts in FactoryBot.

```ruby
FactoryBot.define do
  factory :user do
    sequence(:name) { |i| "username#{i}" }

    after(:create) do |user, evaluator|
      create_list(:post, 2, user: user)
    end
  end
end
```

By using `after(:create)`, a User and Posts that are associated with it are created. Let's use what we defined to write a test for `User#posts_ordered_by_popularity`, a method that returns a User's Posts ordered by the most popular ones first.

```ruby
RSpec.describe User, type: :model do
  describe '#posts_ordered_by_popularity' do
    let!(:user) { create(:user) }
    let!(:post_popular) do
      post = user.posts[0]
      post.update(popularity: 5)
      post
    end
    let!(:post_not_popular) do
      post = user.posts[1]
      post.update(popularity: 1)
      post
    end

    it 'returns posts ordered by popularity' do
      expect(user.posts_ordered_by_popularity).to eq [post_popular, post_not_popular]
    end
  end
end
```

This code is pretty difficult to understand. In this test, the User being created depends on the 2 Posts being created as well. Also, because [update](#dont-change-data-with-update) is being used to change the data, you can't really tell what state the record is in.

To avoid this, first of all let's change the code that created the has many association record as a default.

```ruby
FactoryBot.define do
  factory :user do
    sequence(:name) { |i| "username#{i}" }

    trait(:with_posts) do
      after(:create) do |user, evaluator|
        create_list(:post, 2, user: user)
      end
    end
  end
end
```

By using `trait`, Post isn't created as a default value. You can use any value, so whenever you want a Post from the original association, all you have to do is create a User with `trait`.

Let's make the original association clear inside the test.

```ruby
RSpec.describe User, type: :model do
  describe '#posts_ordered_by_popularity' do
    let!(:user) { create(:user) }
    let!(:post_popular) { create :post, user: user, popularity: 5 }
    let!(:post_not_popular) { create :post, user: user, popularity: 1 }

    it 'returns posts ordered by popularity' do
      expect(user.posts_ordered_by_popularity). to eq [post_popular, post_not_popular]
    end
  end
end
```

Now we've made the test look a lot better.

## Writing tests that use `Time`

When writing tests for time, it's best to use a relative time in relation to the current time when the target of the test's time is not definite. Doing things this way increases the chances of discovering bugs.

As an example, we'll write a scope that gets posts that were published last month, and we'll write it with a definite time.

```ruby
class Post < ApplicationRecord
  scope :last_month_published, -> { where(publish_at: (Time.zone.now - 31.days).all_month) }
end
```

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:april_1st) { create :post, publish_at: Time.zone.local(2017, 4, 1) }
    let!(:april_30th) { create :post, publish_at: Time.zone.local(2017, 4, 30) }

    before do
      create :post, publish_at: Time.zone.local(2017, 5, 1)
      create :post, publish_at: Time.zone.local(2017, 3, 31)
    end

    it 'returns published posts from last month' do
      Timecop.travel(2017, 5, 6) do
        expect(Post.last_month_published).to contain_exactly(april_1st, april_30th)
      end
    end
  end
end
```

This test will always pass, but it has a bug in it.

Let's change it to a relative time.

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:now) { Time.zone.now }
    let!(:last_beginning_of_month) { create :post, publish_at: 1.month.ago(now).beginning_of_month }
    let!(:last_end_of_month) { create :post, publish_at: 1.month.ago(now).end_of_month }

    before do
      create :post, publish_at: now
      create :post, publish_at: 2.months.ago(now)
    end

    it 'returns published posts from last month' do
      expect(Post.last_month_published).to contain_exactly(last_beginning_pof_month, last_end_of_month)
    end
  end
end
```

This test, for example, will fail on May 1st. You won't always be able to detect bugs, but by using CI tests, you can decrease the chances of a bug sticking around for a long time.

## Inserting time from an outer source

[(Japanese) Cookpad Developer Blog - Concerning the scheme and implementation of it when inserting "Current Time"](http://techlife.cookpad.com/entry/2016/05/30/183947)

## Differentiating `before` and `let`(`let!`)

When creating an object (or record) for the basis of tests, `let`(`let!`) and `before` are used.

For example there is a `scope` that has a  'User' model.

```ruby
class User < ApplicationRecord
  scope :active, -> { where(deleted: false).where.not(confirmed_at: nil) }
end
```

If you write this test with `let!` only, it will turn out like this:

```ruby
require 'rails_helper'

Rspec.describe User, type: :model do
  describe '.active' do
    let!(:active) { create :user, deleted: false, confirmed_at: Time.zone.now }
    let!(:deleted_but_confirmed) { create :user, deleted: true, confirmed_at: Time.zone.now }
    let!(:deleted_and_not_confirmed) { create :user, deleted: true, confirmed_at: nil }
    let!(:not_deleted_but_not_confirmed) { create :user, deleted: false, confirmed_at: nil }

    it 'returns active users' do
      expect(User.active).to eq [active]
    end
  end
end
```

If the test is written with both `let!` and `before` it will turn out like this:

```ruby
require 'rails_helper'

RSpec.describe User, type: :model do
  describe '.active' do
    let!(:active) { create :user, deleted: false, confirmed_at: Time.zone.now }

    before do
      create :user, deleted: true, confirmed_at: Time.zone.now
      create :user, deleted: true, confirmed_at: nil
      create :user, deleted: false, confirmed_at: nil
    end

    it 'returns active users' do
      expect(User.active).to eq [active]
    end
  end
end
```

The latter test makes it easier to tell the difference between the object which is the return value, and everything else.

※Some people might think that by adding a name to your code like `let!(:deleted_but_confirmed)` will make things easier to understand, but if the record needs to be named, just writing a comment should suffice.


## Be reserved when keeping code DRY

Some might think that making things DRY is always the best idea, but that's not the case. For example, when making duplicated code abstract by processing it all in one group, depending on the situation and the way it was made abstract, there might be a higher cost than code that was originally decreased by DRY.

### Think before using `shared_examples`

You can deleted duplicated code by using `shared_examples`, but the way it's written can decrease the readability of the code.

As an example, well use `shared_examples` to write a test for the method `Point#increase_by_day_of_the_week` which increases points only by day (of the week) which was passed as an argument. Let's define `shared_examples` in another file, and first just look at the code which will use the `shared_examples`.

```ruby
RSpec.describe Point, type :model do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    it_behaves_like 'point increasing by day of the week', 100 do
      let(:wday) { 0 }
    end

    it_behaves_like 'point increasing by day of the week', 50 do
      let(:wday) { 1 }
    end

    it_behaves_like 'point increasing by day of the week', 30 do
      let(:wday) { 2 }
    end

    # ...
  end
end
```

It's not so easy to understand what's the expected outcome by the conditions set beforehand just by looking at this.

`shared_examples` is defined as follows.

```ruby
RSpec.shared_examples 'point increasing by day of the week' do |expected_point|
  it "increases by #{expected_point}" do
    expect(point.point).to eq 0
    point.increase_by_day_of_the_week(wday)
    expect(point.point).to eq expected_point
  end
end
```

One reason why this test is difficult to read is because the conditions to be set beforehand to create the `shared_examples` are too many.

- `point`, the main object to be tested
- `wday`, the argument to be passed to the method
- `expected_point`, the result to be expected

Also, another reason is because defining each value is a scattered process

- `let(point)` is defined externally
- `it_behaves_like` is defined through `let(wday)` inside a block
- The second argument of `it_behaves_like` is (`expected_point`)

First of all, let's incorporate suitable names to increase the readability of the code.

```ruby
RSpec.shared_examples 'point increasing by the day of the week' do |expected_point:|
  it "increases by #{expected_point}" do
    expect(point.point).to eq 0
    point.increase_by_day_of_the_week(wday)
    expect(point.point).to eq expected_point
  end
end

RSpec.describe Point, type: :model do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    context 'on sunday' do
      let(:wday) { 0 }
      it_behaves_like 'point increasing by day of the week', expected_point: 100
    end

    context 'on monday' do
      let(:wday) { 1 }
      it_behaves_like 'point increasing by day of the week', expected_point: 50
    end

    context 'on tuesday' do
      let(:wday) { 2 }
      it_behaves_like 'point increasing by day of the week', expected_point: 30
    end

    # ...
  end
end
```

By creating a new `context`, we added an explanation about `wday`. Then, by using `expected_point` as a keyword for the expected result, we've given a name to the literal integer passed as an argument, making it clear as to what the number is supposed to represent.

But should is this a case where `shared_examples` should be used in the first place? If we write the test without `shared_examples`, it'll turn out like this:

```ruby
RSpec.describe Point, type: :model, do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    context 'on sunday' do
      let(:wday) { 0 }

      it "increases by 100" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 100
      end
    end

    context 'on monday' do
      let(:wday) { 1 }

      it "increases by 50" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 50
      end
    end

    context 'on tuesday' do
      let(:wday) { 2 }

      it "increases by 30" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 30
      end
    end

    #...
  end
end
```

The more the conditions set beforehand and the arguments themselves increase by using `it_behaves_like`, the more complicated the code gets. The merits of being DRY and discerning whether it exceeds the complexity of the code need to be considered carefully.

## Consider using scopes carefully
### Don't place test data outside of `describe`

For example, let's look at the following spec test:

```ruby
describe 'sample specs' do
  context 'a' do
    # ...
  end

  context 'b' do
    let!(:need_in_b_and_c) { ... }
    # ...
  end

  context 'c' do
    let!(:need_in_b_and_c) { ... }
    # ...
  end
end
```

In this situation, b and c have the same conditions set beforehand, so there might be some people who would like to make this DRY in one level above.

```ruby
describe 'sample specs' do
  let!(:need_in_b_and_c) { ... }        

  context 'a' do
    # ...
  end

  context 'b' do
    # ...
  end

  context 'c' do
    # ...
  end
end
```

But actually this isn't good. This is because there are conditions set beforehand that aren't needed in context 'a'. By using `let!` like this, when there are 10 or even 30 contexts, it's difficult to tell which `let!` corresponds to which context. By now I'm sure you can feel the fear that can come from grouping things together poorly.

Of course, if there are conditions that are used between all contexts, there's not problem in grouping them all together.

### Rules when placing preconditions in each block

- For the preconditions of each block, only write things that will be used for the expectations under them
- Write preconditions inside the expectation only when it is a special case for that expectation

### Other things to be aware of
- The next case is an exception concerning the rules when placing preconditions in each block
- However, it is usually best to define the precondition within the block

```ruby
let!(:user) { create :user, enabled: enabled }

context 'when user is enabled' do
  let(:enabled) { true }
  it { ... }
end

context 'when user is disabled' do
  let(:enabled) { false }
  it { ... }
end

```


## Don't create records that aren't needed

As far as performance, it's best to not create a record when the test works fine without one.

```ruby
describe 'posts#index' do
  context 'when visit /posts' do
    let!(:posts) { create_list :post, 100 }

    before { visit posts_path　}

    it 'displays all post titles' do
      posts.each do |post|
        expect(page).to have_content post.title
      end
    end
  end
end
```

Even with model unit tests, there are a lot of cases where a lot of unneeded records are created.

```ruby
describe 'User' do
  describe '#fullname' do
    let!(:user) { create :user, first_name: 'Shinichi', last_name: 'Maeshima' }

    it 'returns full name' do
      expect(user.fullname).to eq 'Shinichi Maeshima'
    end
  end
end
```

`User#fullname` is a method that isn't affected by a record being saved or not. In this case, `build` (or `build_stubbed`) should be used instead of `create`.

```ruby
describe 'User' do
  describe '#fullname' do
    let!(:user) { build :user, first_name: 'Shinichi', last_name: 'Maeshima' }

    it 'returns full name' do
      expect(user.fullname).to eq 'Shinichi Maeshima'
    end
  end
end
```

You can use `User.name` in a simple case like this.

## Don't change data with update

It's difficult to grasp the final condition of a column of a record created with FactoryBot when it is changed with `update`, and it's also hard to tell which attributes the test depends on, so it's best to avoid this.

```ruby
describe Post do
  let!(:post) { create :post }

  describe '#published?' do
    subject { post.published? }

    context 'when the post has already been published' do
      it { is_expected.to eq true }
    end

    context 'when the post has not been published' do
      before { post.update(publish_at: nil) }

      it { is_expected.to eq false }
    end

    context 'when the post is closed' do
      before { post.update(status: :close) }

      it { is_expected.to eq false }
    end

    context 'when the title includes "[WIP]"' do
      before { post.update(title: '[WIP]hello world')}

      it { is_expected.to eq false }
    end
  end
end
```

Can you tell immediately what attribute depends on the method 'Post#published?'? `update` is mainly used to set and slightly change the "the most used structure of the data being used" as the default value in FactoryBot.

As written in [Default values in FactoryBot](#default-values-in-factorybot), it is good to write default values randomly, without using update.

## Don't overwrite `let`

If you overwrite the parameters defined in `let` inside a context, it is difficult to grasp the final condition of the record so it's best to avoid this, as explained in [Don't change data with update](#dont-change-data-with-update)

```ruby
describe Post do
  let!(:post) { create :post, title: title, status: status, publish_at: publish_at }
  let(:title) { 'hello world' }
  let(:status) { :open }
  let(:publish_at) { Time.zone.now }

  describe '#published?' do
    subject { post.published? }

    context 'when the post has already been published' do
      it { is_expected.to eq true }
    end

    context 'when the post has not been published' do
      let(:publish_at) { nil }

      it { is_expected.to eq false }
    end    

    context 'when the post is closed' do
      let(:status) { :close }

      it { is_expected.to eq false }
    end    

    context 'when the title includes "[WIP]"' do
      let(:title) { '[WIP]hello world'}

      it { is_expected.to eq false }
    end    
  end    
end
```

## Things to watch out for when using `subject`

`subject` is useful when writing an expectation on one line using `is_expected` or `should`, but there are also cases where it can have a negative effect on readability.

```ruby
describe 'ApiClient#save_record_from_api' do
  let!(:client) { ApiClient.new }
  subject { client.save_record_from_api(params) }

  #
  # ...A bunch of expectations...
  #

  context ' when pass { limit: 10 }' do
    let(:params) { { limit: 10 } }

    it 'returns ApiResponse object' do
      is_expected.to be_an_instance_of ApiResponse
    end

    it 'saves 10 items' do
      expect { subject }.to change { Item.count }.by(10)
    end
  end
end
```

In cases like this, it's not very easy to tell what  the `subject` of `expect { subject }` is doing, so you have to go all the way to the top of the file to figure out what the `subject` is.

"subject" is a noun, so placing it where an effect is expected will just confuse the reader.

When `is_expected` is used with an implicit `subject`, and an explicitly defined `subject` are both mixed in the same code, if you really want to use `subject` in this case, it's good to give subject a name.

```ruby
describe 'ApiClient#save_record_from_api' do
  let!(:client) { ApiClient.new }
  subject(:execute_api_with_params) { client.save_record_from_api(params) }

  context 'when pass { limit: 10 }' do
    let(:params) { { limit: 10 } }

    it 'returns ApiResponse object' do
      is_expected.to be_an_instance_of ApiResponse
    end

    it 'saves 10 items' do
      expect { execute_api_with_params }.to change { Item.count }.by(10)
    end
  end
end
```

This is a lot easier to understand than when it was `expect { subject }`

When `is_expected` isn't being used, it's good forget using `subject` and just write `client.save_record_from_api(params)` inside each expectation.

## Avoid using `allow_any_instance_of`

It's also written in the [official documentation](https://relishapp.com/rspec/rspec-mocks/docs/working-with-legacy-code/any-instance), but there is a chance that the test's target design will bug out when using `allow_any_instance_of` (`expect_any_instance_of`).

As an example, Let's write a test for `Statement#issue`

```ruby
class Statement
  def issue(body)
    client = TwitterClient.new
    client.issue(body)
  end
end
```

```ruby
describe Statement do
  describe '#issue' do
    let!(:statement) { Statement.new }

    it 'calls TwitterClient#issue' do
      expect_any_instance_of(TwitterClient).to receive(:issue).with('hello')
      statement.issue('hello')
    end
  end
end
```

The reason we used `expect_any_instance_of` was to combine the `Statement` class and the `TwitterClient` class. Let's loosen the relation.

```ruby
class Statement
  def initialize(client: TwitterClient.new)
    @client = client
  end

  def issue(body)
    client.issue(body)
  end

  private

  def client
    @client
  end
end
```

```ruby
describe Statement do
  describe '#issue' do
    let!(:client) { double('client') }
    let!(:statement) { Statement.new(client: client) }

    it 'calls TwitterClient#issue' do
      expect(client).to receive(:issue).with('hello')
      statement.issue('hello')
    end
  end
end
```

The code has been fixed so any class or object that has `issue` as a method will be considered a `client`. By specifying `client` externally, the code can except any other clients added in the future such as `FacebookClient`. The relationship between the two classed has been loosened, and we were able to write a test with just a simple mock object.
