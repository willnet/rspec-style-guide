# RSpec Style Guide
## What's this?

Etiquette for writing test code with high readibility

We would like to improve our style guide by hearing all of your opinions, so please open issues and pull requests that correspond to the following:

- Doubts or questions
- RSpec etiquette that isn't written here
- The content is fine but the way it's expressed is strange
- Better sample code

# Presumed knowledge

- RSpec
- FactoryGirl

## `describe` and `context`

`describe` and `method` are the same methods, but they can be used in the following ways to help differentiate the type of code your testing.

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
        expect { stack.push(nil) }.to raises_error(ArgumentError)
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
