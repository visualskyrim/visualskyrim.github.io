---
layout: post
title: "Test your Redux containers with Enzyme"
description: "
"
modified: 2017-01-26
tags: [react, jquery]
category: Snippets
image:
  feature: abstract-7.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

> React with Redux has become a very common stack these days.
People with very little background of front-end, like me, can write something that turns out not bad.
> However, testing a React component connected to the Redux can sometimes be painful.

In this post, I want to share some of the ideas about how to test against a Redux container.

-

First of all, we need a container to be tested against.

In the following gist, we have a text box that:

- saves the value of input into component’s local state
- submits the input value via Redux action dispatch to the Redux store.
- shows the input value in the Redux store.



{% highlight js %}
import React from 'react'
import PropTypes from 'prop-types'
import { connect } from 'react-redux'
import { submitValue } from '../store/modules/showBox'

export class ShowBox extends React.Component {
  constructor(props) {
    super(props)
    this.state = {
      searchString: this.props.search || ""
    }
  }

  handleInput = (event) => {
    this.setState({
      searchString: event.target.value
    })
  }

  render () {
    return (
      <form onSubmit={(e) => this.props.handleShowSubmit(this.state.searchString, e)}>
        <div>
          <input
            type="search"
            className="form-control"
            placeholder="Search"
            value={this.state.searchString}
            onChange={this.handleInput}
          />
          <div>
            <i className="icon-search"></i>
          </div>
        </div>
      </form>
    )
  }
}

export default connect(
  (state) => ({
    search: state.showBox.search,
  }),
  (dispatch) => {
    return {
      handleShowSubmit: (text, e) => {
        if (e) {
          // Avoid redirecting
          e.preventDefault()
        }
        dispatch(submitValue(text))
      }
    }
  }
)(ShowBox);
{% endhighlight %}

> Note this component is only used for this example. Separating the text box and the representative part is more practical in a real project.


I highly recommend using [Enzyme](http://airbnb.io/enzyme/index.html) with all your React components’ tests. This utility makes it much easier to traverse and assert your React/Redux components.

Besides, it now works pretty well with Mocha and Karma and other stuff you might use in your test.


## Redux store wrapper for the tests

One of the painful parts of testing a Redux container is that we can’t just render a container and then evaluate the DOM anymore. That’s because container will ask for the Redux store in [mapStateToProps()](http://redux.js.org/docs/basics/UsageWithReact.html#implementing-container-components) when you connect your component to Redux, and without it, test fails.

So, before we start to write testing code, we will need a “Redux wrapper” for all of our testing components to provide Redux store, which is something pretty much like [what we have usually done](https://github.com/reactjs/react-redux/blob/master/docs/api.md#provider-store) to our root component in the application.

Luckily, Enzyme provides a [shallow-render function](http://airbnb.io/enzyme/docs/api/shallow.html#arguments) that can take a context as second parameter, where we can put a Redux store for our tests.


{% highlight js %}

import { shallow } from 'enzyme';

const shallowWithStore = (component, store) => {
  const context = {
    store,
  };
  return shallow(component, { context });
};

export default shallowWithStore;
{% endhighlight %}


## Make sure it’s working

Let’s first try to shallow-render our component in the tests with the ``shallowWithStore`` we created. We are going to mock our Redux store by using [createMockStore(state)](https://www.npmjs.com/package/redux-test-utils) from [redux-test-utils](http://redux-test-utils/).

{% highlight js %}
describe('ConnectedShowBox', () => {
  it("should render successfully if string is not provided by store", () => {
    const testState = {
      showBox: {}
    };
    const store = createMockStore(testState)
    const component = shallowWithStore(<ConnectedShowBox />, store);
    expect(component).to.be.a('object');
  });
});
{% endhighlight %}


By using ``shallowWithStore`` with the mocked store, we can finally shallow-render our container.

{% highlight js %}
it("should render a text box with no string inside if search string is not provided by store", () => {
  const testState = {
    showBox: {
      search: ""
    }
  };
  const store = createMockStore(testState)
  const component = shallowWithStore(<ConnectedShowBox />, store);
  expect(component).to.be.a('object');


  expect(component.dive().find("").prop("value")).to.equal("")


  component.dive().find("").simulate("change", { target: { value: "Site" } });
  expect(component.dive().find("d").prop("value")).to.equal("Site")
});
{% endhighlight %}

To test the DOM structure, you can use [find()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/find.html) to traverse the rendered result, then use [prop()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/prop.html) to get the desired attribute then make the assertion.
To test event handler, you can first get the element with find(), then use simulate() to trigger the event. After that, just make assertion as usual.
Please note that we call dive() before calling find().

## Test non-Redux feature with Enzyme

There are two non-Redux features we might want to test:

- DOM structure.
- handleInput function in the component.

Achieving these two goals is extremely simple with the Enzyme.

{% highlight js %}
it("should render a text box with no string inside if search string is not provided by store", () => {
  const testState = {
    showBox: {
      search: ""
    }
  };
  const store = createMockStore(testState)
  const component = shallowWithStore(<ConnectedShowBox />, store);
  expect(component).to.be.a('object');


  expect(component.dive().find("").prop("value")).to.equal("")


  component.dive().find("").simulate("change", { target: { value: "Site" } });
  expect(component.dive().find("d").prop("value")).to.equal("Site")
});
{% endhighlight %}

To test the DOM structure, you can use [find()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/find.html) to traverse the rendered result, then use [prop()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/prop.html) to get the desired attribute then make the assertion.

To test event handler, you can first get the element with ``find()``, then use [simulate()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/simulate.html) to trigger the event. After that, just make assertion as usual.

Please note that we call ``dive()`` before calling ``find()``.

> I used [Chai](http://chaijs.com/) for assertion. It might look different from what you would write depending on your setup.


One thing that should be careful about is that if we shallow-render a connected container, the result of rendering will not reach the DOM inside the component.

This means the whole DOM inside ShowBox will not be rendered, which will cause following assertion to fail:

{% highlight js %}
expect(component.find("").prop("value")).to.equal("");
{% endhighlight %}


The reason is that shallow-render ***only renders the direct children of a component***.

When we shallow-render connected container ``<ConnectedShowBox />``, the deepest level it gets rendered is ``<ShowBox />``. All the stuff inside the ShowBox’s ``render()`` will not be rendered at all.

To make the result of shallow-render go one layer deeper( to render the stuff inside the `<ShowBox />`), we need to call [dive()](http://airbnb.io/enzyme/docs/api/ShallowWrapper/dive.html) first.

> You can also test mapStateToProps here by simply changing the mocked ``state(testState)``.


## Test Redux dispatch mapDispatchToProps

Test ``mapDispatchToProps`` with Enzyme is really easy.

{% highlight js %}
it("should render a text box with no string inside if search string is not provided by store", () => {
  const testState = {
    showBox: {
      search: ""
    }
  };
  const store = createMockStore(testState)
  const component = shallowWithStore(<ConnectedShowBox />, store);

  component.dive().find("form > div > input").simulate("change", { target: { value: "Site" } });
  expect(component.dive().find("d").prop("value")).to.equal("Site")


  component.dive().find("form").simulate("submit");
  expect(store.isActionDispatched({
    type: "showBox/SUBMIT",
    searchString: "Site"
  })).to.be.true;
});
{% endhighlight %}

Just like what we did to test ``handleInput()``. First we use ``find()`` to get the element that we need to trigger, then use the ``simulate()`` to trigger the event.

Finally use ``isActionDispatched()`` to check whether the expected action is dispatched.


-

And that is it! I think this is enough for most the use cases in testing.

Here is some further challenges for this topic:

- Make it work with code coverage utilities.
- Make it work with GraphQL(Apollo)
