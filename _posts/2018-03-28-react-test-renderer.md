---
layout: post
title:  Using the React Test Renderer to write unit tests
---

# Using the React Test Renderer to write unit tests

It's not all that obvious how we should be writing unit tests for our React components. There are plenty of options for testing our components:

* [Jest](https://facebook.github.io/jest) supports snapshot tests
* [enzyme](http://airbnb.io/enzyme/) is a full-featured component testing library
* [react-testing-library](https://github.com/kentcdodds/react-testing-library) is a layer on top of React's included test utilities

But what if you're wary of adding _yet another JavaScript dependency_ to test your React components? What if you only want to use React's own tooling?

This guide explains how to write component unit tests using **only React's included test utilities**.

To make executing our tests easier, this examples in this guide are going to use [Jest](https://facebook.github.io/jest) as a test runner, so if you're following along, install it like so:
```bash
npm install -D jest
```

### Test utilities included with React

React comes with several test utilities:
* [The full test renderer](https://reactjs.org/docs/test-renderer.html) which we'll be covering. Enables you to fully render your components and make assertions about the markup, all without using an actual DOM.
* [The shallow test renderer](https://reactjs.org/docs/shallow-renderer.html) which enables you to make assertions about the _first level_ of your components' markup
* [The test utils](https://reactjs.org/docs/test-utils.html), a suite of tools for writing component tests

Taken together, these tools enable us to create robust component unit tests. They don't have all of the features of a library like [enzyme](http://airbnb.io/enzyme/), but they're also light-weight, officially supported, and aren't authored by a third party.

This guide will explain how to use the **full test renderer (`react-test-renderer`)** and **the test utils (`ReactTestUtils`)** to write component tests.

### Installing the tools

The **test utils** come with the [`react-dom`](https://www.npmjs.com/package/react-dom) package (maintained and publish by the React team), so we'll need to install `react-dom` (a very common dependency that's probably already a dependency of our project):
```bash
npm install react-dom
```

The **test renderer** is included in a separate package (also maintained by the React team): [`react-test-renderer`](https://www.npmjs.com/package/react-test-renderer):
```bash
npm install -D react-test-renderer
```

### Test renderer: basic usage

First we need a component to test. Let's say we're trying to test a multiple choice test component. It should render a question, some choices, and the selected answer:

```javascript
import React from 'react'
import PropTypes from 'prop-types'

export default class MultipleChoice extends React.Component {
  state = {
    // Represents the currently selected choice.
    // It's falsy if there's nothing selected.
    selectedChoice: undefined,
  }

  // Just updates our selectedChoice to whatever is passed in.
  onClickChoice (choice) {
    this.setState({
      selectedChoice: choice,
    })
  }

  render () {
    return (
      <div>
        <h3>{ this.props.question }</h3>
        <ul>
          {
            // Give each choice a <li>, highlighting the selected choice.
            Object.keys(this.props.choices).map(choice => (
              <li key={choice} style={{
                background: choice === this.state.selectedChoice
                  ? '#81ff98' // Just some color, picked arbitrarily
                  : 'transparent',
              }}>
                <label>
                  <input type='checkbox'
                         // The checkbox of the selected choice should be checked
                         checked={choice === this.state.selectedChoice}
                         // Updates the selection when a choice is clicked
                         onClick={() => this.onClickChoice(choice)} />
                  <strong>{ choice }</strong>
                </label>
                <p>{ this.props.choices[choice] }</p>
              </li>
            ))
          }
        </ul>

        { (typeof this.state.selectedChoice === 'string') && (this.state.selectedChoice in this.props.choices) && (
          <p>
            You've selected <strong>{ this.state.selectedChoice }</strong>
          </p>
        )}
      </div>
    )
  }
}

MultipleChoice.propTypes = {
  // The "node" PropType is anything that can be rendered as a React element
  choices: PropTypes.objectOf(PropTypes.node).isRequired,

  question: PropTypes.node.isRequired,
}
```
