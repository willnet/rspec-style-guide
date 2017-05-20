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
- FactoryGirl

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

# Default values in FactoryGirl

When using FactoryGirl, you can create default values for the columns of each of your models. It's good to create random variables for each column when you do this. Also, by explicitly defining only which values are necessary inside the tests, it makes it easier to understand which values are the most important.

### A bad example

For example, we have a column named `active` which will determine whether a user account is active or not. You will often see cases like this where the active/inactive column is fixed.

```ruby
FactoryGirl.define do
  factory :user do
    name 'willnet'
    active true
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

### Good example

```ruby
FactoryGirl.define do
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
