## How to avoid tying tests to implementation

### Suggestion: test behavior, not implementation

Tests are one of the most important parts of a codebase. They can catch many bugs before they are released. They can ensure your programs work as expected.

But, tests tied to implementation details can bind you to how code was written in unhelpful ways.

Testing behavior means testing **what code does**. Testing implementation means testing **how code works**.

#### Testing behavior

When you write tests for behavior (the "what"), you can refactor your code (the implementation) in confidence without breaking the tests; the tests only break when behavior has changed (unintentionally - which means you've introduced a bug - or intentionally - which means tests need to be updated to the new behavior).

These types of tests are extremely valuable because a broken test means your code is broken.

Testing behavior guarantees your code works as expected (for the behaviors you have tested).

#### Testing implementation

When you write tests for implementation (the "how"), tests break when you change code, even if the behavior didn't change.

These types of tests aren't very valuable because a broken test doesn't mean your code is broken and a passing test doesn't mean your code works as expected.

Testing implementation only guarantees your code was written as it was tested.

### Examples

Let's look at some real examples to better understand the difference between the two.

_Although the examples are in JavaScript/React with Jest, the principles apply to any programming language, test framework etc. Also, the examples are not hard rules to be taken as law but general guidelines._

#### Contrived example

Before:

```javascript
import lodash from "lodash";
import add from "./add";

let addSpy;

beforeEach(() => {
  addSpy = jest.spyOn(lodash, "add");
});

afterEach(() => {
  addSpy.mockReset();
  addSpy.mockRestore();
});

test("takes two numbers and adds them together with lodash add", () => {
  const sum = add(5, 10);
  expect(addSpy).toHaveBeenCalledWith(5, 10);
});
```

This test is tied to implementation because it relies on the `add` function using `lodash` underneath the hood. It also has hidden side effects.

After:

```javascript
import add from "./add";

test("can add", () => {
  const input = add(5, 10);
  const output = 15;
  expect(input).toBe(output);
});
```

Now this test only checks behavior. The implementation of `add` can be changed in confidence without breaking the test. The test is also now self-contained so changes can be made to it without breaking other tests.

Ok, now that we got that contrived example out of the way...here are some more realistic examples.

#### Replacing implementation with behavior in test labels

Before:

```javascript
test('renders <UtilityNav /> with props'...
```

This test label relies on implementation with the name of "UtilityNav" and React "props".

After:

```javascript
test('can dismiss the onboarding guide'...
```

Now this test label focuses on the behavior we are checking not specific to file structure/libraries etc.

#### Replacing setup/teardown with explicit self contained tests

Before:

```jsx
import React from "react";
import { shallow } from "enzyme";
import { mockStripeKey, unmockStripeKey } from "fixtures/stripeKey";
import PromoCode from "./PromoCode";
import Testimonials from "./Testimonials";
import { Signup } from ".";

let component;
let stripeMetaTag;
const props = {
  cardHolderName: "Jane Doe",
  cardNumber: "4242424242424242"
};

beforeEach(() => {
  component = shallow(<Signup {...props} />);
  stripeMetaTag = mockStripeKey("stripe-key");
});

afterEach(() => {
  unmockStripeKey(stripeMetaTag);
});

test("renders a Testimonials component", () => {
  expect(component.find(Testimonials).exists()).toBe(true);
});

describe("when displayDiscount is true", () => {
  beforeEach(() => {
    const discountProps = {
      ...props,
      displayDiscount: true
    };
    component = shallow(<Signup {...discountProps} />);
  });

  test("renders a PromoCode", () => {
    expect(component.find(PromoCode).exists()).toBe(true);
  });
});
```

The difficulty with this approach is that when looking at a test like `renders a PromoCode`, to debug and make changes you have to look at multiple levels of inheritance and mutations.

After:

```jsx
import React from "react";
import { mount } from "enzyme";
import { mockStripeKey, unmockStripeKey } from "fixtures/stripeKey";
import { Signup } from ".";

test("has testimonials", () => {
  const stripeMetaTag = mockStripeKey("stripe-key");

  const signup = mount(
    <Signup cardHolderName="Jane Doe" cardNumber="4242424242424242" />
  );

  const input = signup.text();
  const output = "This is one of my favorite apps";
  expect(input).toContain(output);

  unmockStripeKey(stripeMetaTag);
});

test("can use a promo code when enabled", () => {
  const stripeMetaTag = mockStripeKey("stripe-key");

  const signup = mount(
    <Signup
      cardHolderName="Jane Doe"
      cardNumber="4242424242424242"
      displayDiscount
    />
  );

  const input = signup.find('[data-testid="promoCode"]').exists();
  const output = true;
  expect(input).toBe(output);

  unmockStripeKey(stripeMetaTag);
});
```

This isn't as "DRY". But I've found having explicit tests makes it much easier to update tests without breaking things because tests are self-contained. Having explicit tests like this means tests aren't tied together with hidden state at multiple inheritance levels. Also, the tests now test behavior instead of implementation (other than the minimal required implementation detail of the `data-testid`).

If the repetition is cumbersome to you, you could put the repeating pieces in factory functions so that the output is still pure (has the same output every time you use it, instead of being mutated in inheritance levels).

I also think having some before/after blocks is totally fine as long as you keep it simple (maybe one level deep?). It's when there are multiple levels of inheritance that it becomes difficult to understand and make changes.

#### Replacing type checking tests with a type checker

Before:

```javascript
expect(typeof startDateInput.prop("onChange")).toBe("function");
```

These kinds of assertions in tests are easier and have better deep checking with static analysis tools like type checkers.

After:

I'd suggest deleting these sorts of type checking tests and instead rely on a type checker (for example, TypeScript or Flow in JavaScript). For example:

```javascript
type Props = {
  onChange: (newValue: DateTime) => void,
  ...
}

const DateInput = (props: Props) => ...
```

#### Removing wire-up/syntax testing

Before:

```jsx
import React from "react";
import { shallow } from "enzyme";

let accountSelect;
let accounts = getPopulatedAppState().accounts.data;

test("renders a <BaseAccountSelect />", () => {
  accountSelect = shallow(
    <AccountSelect accounts={accounts} selectedAccountIds={["1", "2"]} />
  );

  expect(accountSelect.find(BaseAccountSelect).length).toBe(1);
  const props = accountSelect.find(BaseAccountSelect).props();
  expect(props.accounts).toBe(accounts);
  expect(props.selected).toEqual(["1", "2"]);
  expect(props.stackOptions).toBe(true);
});
```

The assertions in these kinds of tests are testing that the following wire-up/syntax in React works:

```jsx
<BaseAccountSelect accounts={accounts} ... />
```

Which is covered by type checking.

Also, style wire up tests like this:

```javascript
test("styles the component as unqueueable", () => {
  expect(body.props("className")).toContain(styles.unqueueable);
});
```

After:

I'd suggest removing these types of tests because they have a high cost (can't make changes to implementation without breaking them) and low value.

I think it would be more valuable to instead test specific behaviors. For example:

```jsx
import React from "react";
import { mount } from "enzyme";

test("shows amount of selected accounts out of the total accounts available to select", () => {
  const accountSelect = mount(
    <AccountSelect
      accounts={{
        data: [
          {
            type: "account",
            id: "1"
          },
          {
            type: "account",
            id: "2"
          }
        ]
      }}
      selectedAccountIds={["2"]}
    />
  );

  const input = accountSelect.text();
  const output = "1/4";
  expect(input).toContain(output);
});
```

#### Breaking out tests unrelated to UI into utility modules

Before:

When we have UI code that has utility logic in it like this with related UI tests:

```javascript
const hasMultipleTwitterAccounts = accounts
  && accounts.filter(
    account => account.platform === 'twitter'
  ).length > 1

...

<OneTwitterAccountPerContentAlert hasMultipleTwitterAccounts={hasMultipleTwitterAccounts}
/>
```

The logic with `hasMultipleTwitterAccounts` doesn't have anything specific to UI in it, but it's tied to React and Enzyme because it is in the component code.

After:

We can pull the tests out of the UI and into their own utility module like this:

```javascript
import hasMultipleTwitterAccounts from ".";

test("returns false when there aren't any accounts", () => {
  const input = hasMultipleTwitterAccounts([]);
  const output = false;
  expect(input).toBe(output);
});

test("returns false when there aren't any twitter accounts", () => {
  const input = hasMultipleTwitterAccounts([
    {
      platform: "facebook"
    }
  ]);
  const output = false;
  expect(input).toBe(output);
});

test("returns false when there is one twitter account", () => {
  const input = hasMultipleTwitterAccounts([
    {
      platform: "twitter"
    },
    {
      platform: "facebook"
    }
  ]);
  const output = false;
  expect(input).toBe(output);
});

test("returns true when there are multiple twitter accounts", () => {
  const input = hasMultipleTwitterAccounts([
    {
      platform: "twitter"
    },
    {
      platform: "facebook"
    },
    {
      platform: "twitter"
    }
  ]);
  const output = true;
  expect(input).toBe(output);
});
```

Then implement the utility module:

```javascript
import type { Account } from "types";

const hasMultipleTwitterAccounts = (accounts: Account[]) => {
  if (!accounts) {
    return false;
  }

  const twitterAccounts = accounts.filter(
    account => account.platform === "twitter"
  );

  return twitterAccounts.length > 1;
};

export default hasMultipleTwitterAccounts;
```

Then replace the logic in the UI with the utility module:

```jsx
import hasMultipleTwitterAccounts from './hasMultipleTwitterAccounts'

...

<OneTwitterAccountPerContentAlert
  hasMultipleTwitterAccounts={hasMultipleTwitterAccounts(accounts)}
/>
```

Now the tests and logic aren't tied to any UI / libraries.

#### Replacing tests tied to DOM structure with output of behaviors

Before:

```jsx
import React from "react";
import { shallow } from "enzyme";
import Badge from "typography/Badge";
import Header from ".";

test("has warning badge when there is an error", () => {
  const header = shallow(<Header errorCount={2} />);
  const badge = result.find(Badge);
  expect(
    badge
      .children()
      .first()
      .children()
  ).toBe("2");
});
```

For example, with React and Enzyme, using `shallow` rendering means you need to traverse through specific DOM structure with `.children`, `.dive`, `.find(a)` etc. This ties you to that DOM structure so if you want to add another wrapper or change the `a` tag to a `button` etc. your tests will break.

After:

```jsx
import React from "react";
import { mount } from "enzyme";
import Header from ".";

test("has warning badge when there is an error", () => {
  const header = mount(<Header errorCount={42} />);
  const input = header.text();
  const output = "42";
  expect(input).toContain(output);
});
```

Now this test isn't tied to DOM structure or specific components etc.

To get around DOM structure testing with React and Enzyme you can use `mount` and methods like `.text()`, `.html()`, `.find('[data-testid="promoCode"]')`, `.find({ someProp: someValue})` etc. In theory, shallow rendering sounds cool but in practice it ties you to implementation.

#### Replacing most spies/mocks with output of behaviors

Before:

```jsx
import React from "react";
import { shallow } from "enzyme";
import moment from "moment";
import { DateCreated } from ".";

let dateCreated;
let filter;
let props;

beforeEach(() => {
  filter = {};
  props = { filter, onFilterPatch: jest.fn() };

  dateCreated = shallow(<DateCreated {...props} />);
});

describe("when the input value changes", () => {
  describe("for the start date", () => {
    let startDateInput;

    describe("by default", () => {
      let newStartDate;

      beforeEach(() => {
        startDateInput = dateCreated.find({ placeholder: "Pick start date" });
        newStartDate = moment("2017-09-24");
        startDateInput.simulate("change", newStartDate);
      });

      test("invokes onChange with a formatted startDate", () => {
        const formattedStartDate = newStartDate.format("YYYY-MM-DD");
        expect(props.onFilterPatch).toHaveBeenCalledWith({
          startDate: formattedStartDate
        });
      });
    });
  });
});
```

After:

```jsx
import React from 'react'
import { mount } from 'enzyme'
import { createWaitForElement } from 'enzyme-wait'
import moment from 'moment'
import { DateCreated } from '.'

test('can change the start date', () => {
  const dateCreated = mount(<DateCreated filter={{}} />)
  const changedStartDatetime = moment('2017-09-24')
  const startDateInput = dateCreated.find('[data-testid="dateInput"]')

  startDateInput.simulate('click')
  await createWaitForElement('[data-testid="openDateInput"]')(dateCreated)
  startDateInput.simulate('change', changedStartDatetime)
  await createWaitForElement('[data-testid="closedDateInput"]')(dateCreated)

  const input = dateCreated.text()
  const output = '2017-09-24'
  expect(input).toContain(output)
})
```

Now the test checks behavior (waiting on async React state changing) instead of function passing implementation with spies.

However, I'm not saying all spies are bad. If your behavior you are testing is that a function is called, a spy can be useful.

#### Moving tests with many side effects to end-to-end tests

If it is impossible to write a test with simple input => output, end-to-end tests might be a better fit. A few simple end-to-end tests can provide huge coverage for little cost.

Here is one example:

```javascript
describe("logout", () => {
  test("can logout", async () => {
    await page.waitForSelector('[data-testid="userMenuButton"]');
    await page.click('[data-testid="userMenuButton"]');
    await page.waitForSelector('[data-testid="userMenuOpen"]');
    await page.click('[data-testid="logoutLink"]');
    await page.waitForSelector('[data-testid="userLoginForm"]');
  });
});
```

These types of tests can remove the need for many heavily mocked and brittle unit tests trying to accomplish the same thing. They also ensure these integrated pieces work together in a real environment which is very valuable (because you will be notified if any of the pieces stop working - ie your database or API is down or a button no longer works etc.).

### Indicators

Here are some good indicators that you are testing behavior instead of implementation:

- You can write tests for your code without looking at how the code is written
- You can change how your code is written (unrelated to behavior) without breaking your tests

### Bottom line

The moral of the story is: each test should focus on a single behavior you don't want to break :)
