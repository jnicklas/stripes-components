# Testing in Stripes Components

Tests can fortify new features from breaking changes, help prevent
bugs from being reintroduced, and overall future-proof against an
ever-changing ecosystem. As more people get involved and contribute,
they can be confident in fixing or introducing changes without
breaking existing functionality.

When we write tests, we need to make sure that they are resilient to
change as well as being easily understandable should somebody need to
add or adjust tests as other features are introduced and bugs are
fixed.

## Directory structure

Each component's directory should contain a `tests` directory that
contains test files named with the component name, and sometimes a
specific test suite name. The `tests` directory should also contain an
often quite useful `interactor.js` file. For example, the directory
structure for `<Button>` looks like this:

```
Button
|-Button.js
|-Button.css
|-index.js
|-readme
|-tests
|  |-interactor.js
|  |-Button-test.js
```

## When Should I Write Tests?

**Anytime you change or add features, you should be changing or adding
tests.**

If you are making a change to existing functionality, existing tests
should be updated. If there were no existing test to update, tests
should be added. If you are adding new features, you should write
plenty of tests for those new features so that you can rest assured
nobody else (or even yourself) accidentally breaks it in the future.

**Anytime you fix a bug, you should be adding new tests or adding to
existing tests.**

Bugs can obviously be a real pain. Even "simple" bugs can keep popping
up repeatedly as _seemingly unrelated_ things change in the code. When
you fix a bug, you most likely want it fixed for good. Writing tests
or fortifying existing tests makes it easily known if the bug is
accidentally introduced again when said tests fail.

**Write tests first!!!**

If you're writing a feature or fixing a bug, write or change some
tests first so that they fail expectedly, or the bug is successfully
reproduced. Then as you start adding or making changes, and your tests
start passing one-by-one, you know you're headed in the right
direction until they all pass when the feature is complete or the bug
is fixed.

## How Do I Write Tests?

Tests in stripes components are written using
[Mocha](https://mochajs.org/) via
[`@bigtest/mocha`](https://github.com/bigtestjs/mocha). At the core of
this package is the concept of _convergences_ which make testing
asynchronous applications both robust and fast. They accomplish this
by freeing us from the burden of worrying about when a component, or
any updates to a component, will render into the DOM. If a test
initially fails, as long as the component renders within the timeout,
the test will rerun and eventually pass. To read more about how this
strategy works, and why it is so effective, checkout the
[documentation on
convergences](https://github.com/bigtestjs/convergence#why-convergence).

### Getting Started

A good place to start is by looking at existing tests. If the
component you're working on already has tests, you can add new tests
to existing setup hooks, or copy similar sections and change relevant
pieces. If the component does not have existing tests, check out
another component's tests, like [the `Button`
component](https://github.com/folio-org/stripes-components/blob/master/lib/Button/tests/Button-test.js),
and follow the same patterns but specific to the component you're
working on.

The patterns themselves can be outlined in a few simple steps:

1. Do any setup work necessary
2. Perform any actions on the component that we want to verify
3. Make assertions against the desired state

Here's a sample of tests from the `Button` component:

``` javascript
import React from 'react';

// testing tools
import { describe, beforeEach, it } from '@bigtest/mocha';
import { expect } from 'chai';

// test helpers
import { mount } from '../../../tests/helpers';

// the component to test
import Button from '../Button';

// interactor used to perform actions (more on this later)
import ButtonInteractor from './interactor';

// the actual tests
describe('Button', () => {
  const button = new ButtonInteractor();
  let clicked;

  // 1. Do any setup work necessary
  beforeEach(async () => {
    // set to false before each test
    clicked = false;

    // mount our component with props
    await mount(
      <Button onClick={() => { clicked = true; }} id="button-test">
        test
      </Button>
    );
  });

  // make assertions against the initial rendered state

  it('renders a <button> tag', () => {
    expect(button.isButton).to.be.true;
  });

  it('renders child text as its label', () => {
    expect(button.text).to.equal('test');
  });

  it('has an id on the root element', () => {
    expect(button.id).to.equal('button-test');
  });

  // testing a specific action

  describe('clicking the button', () => {

    // 2. Perform any actions on the component that we want to verify
    beforeEach(async () => {
      await button.click();
    });

    // 3. Make assertions against the desired state
    it('calls the onClick handler', () => {
      expect(clicked).to.be.true;
    });
  });
});
```

We use a helper called `mount` in our first `beforeEach` hook. This
helper will clean up and teardown any previously mounted component,
then mount our new component into the DOM for us. We use the
`async`/`await` syntax to wait for React to mount our component so the
tests don't start running too early. Since this helper cleans up
previous components on _the next_ call, this allows us to investigate
and debug our tests after they run. You can then use Mocha's `it.only`
to isolate a specific test to debug.

Since `@bigtest/mocha` makes our tests convergent, and could possibly
run them multiple times, _they must only contain assertions_. You'll
also notice when we perform our `clicked` test that the `await
button.clickButton()` action is separated into it's own `beforeEach`
hook so it isn't accidentally invoked more than once. Separating
side-effects this way and repeatedly running our assertions until
completion, results in _very fast testing times_. For example, the
tests above run and complete in _less than a tenth of a second_.

### Testing Component Interactions

In the previous example, all of our tests reference the `button = new
ButtonInteractor()` at the top of the test suite which is imported
from [the button's `tests/interactor.js`
file](https://github.com/folio-org/stripes-components/blob/master/lib/Button/tests/interactor.js).

``` javascript
import {
  interactor,
  is,
  attribute,
  hasClass
} from '@bigtest/interactor';

import css from '../Button.css';

export default interactor(class ButtonInteractor {
  static defaultScope = `.${css.button}`;
  id = attribute('id');
  href = attribute('href');
  isAnchor = is('a');
  isButton = is('button');
  rendersDefault = hasClass(css.default);
  rendersPrimary = hasClass(css.primary);
  rendersBottomMargin0 = hasClass(css.marginBottom0)
});
```

You should always use an interactor if your tests need to interact
with the DOM either by reading attributes, properties, text,
classnames, or by sending actions such as click, focus, blur, or
change events.

An `Interactor` is a powerful, customizable, composable, [page
object](https://martinfowler.com/bliki/PageObject.html). This adds a
layer of durability to our suite because when things like classnames
or the markup inevitably change, we should only need to update our
interactor as opposed to updating each of our tests. Interactors can
also be composed by each other, so if another component uses a button,
that component's interactor can use the `ButtonInteractor` too.

Interactors are also convergent and will wait for elements to exist in
the DOM before interacting with them. Interactor properties are lazy
and do not query for the element until they are accessed. To learn
more about what they do, how to create your own interactors, and how
to then compose interactors, check out the [BigTest Interactor
Guides](https://bigtestjs.io/guides/interactors/introduction/).

#### Note about CSS classnames

When importing the `css` classnames, properties will return a
_space-separated_ list of names. To query for an element via classname,
you must denote a class using a leading period (`.classname`). So
while `.${css.button}` works when a _single classname_ is returned,
it will not work when using postcss `compose:`, which returns
_multiple classnames_. For this, another helper is needed to convert
the classname string into a proper selector.

``` javascript
import { selectorFromClassnameString } from '../../../tests/helpers';

// neccessary due to composing classnames with postcss `composes`
const btnClass = selectorFromClassnameString(`.${css.button}`);
```

### "I need the `stripes`/`redux`/`intl` context"

**Components should be able to be tested in complete isolation.**

Components are meant to be reusable and composable. By having a
component rely on any context, they lose their reusability and make it
more difficult to compose them without the required context. A prime
example of this are the various `redux-form` reliant field
components. It was _impossible_ to use many of these components
outside of a `redux-form` connected HOC, [until
recently](https://issues.folio.org/browse/STCOM-269).

In fact, React is [deprecating the previously experimental context
API](https://reactjs.org/docs/legacy-context.html) in favor of a new
[props-based component API
instead](https://reactjs.org/docs/context.html). Once Stripes
Components and it's dependencies fully support the new context API,
you should only need to provide additional props instead of requiring
a top-level provider for context.

Until then, however, if you _absolutely need_ to provide your
component with the old context, there exists another testing mount
helper that will set up typical contexts found within a stripes
application.

``` javascript
// instead of importing `mount`, import `mountWithContext`
import { mountWithContext } from '../../../tests/helpers';

// ...

beforeEach(async () => {
  await mountWithContext(
    <Datepicker />
  );
});
```
