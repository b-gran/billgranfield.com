---
layout: post
title:  Using the React Test Renderer to write unit tests
---

# Using the React Test Renderer to write unit tests

This guide explains how to write component unit tests using **React's official test utilities**.

To make executing our tests easier, this examples in this guide use [Jest](https://facebook.github.io/jest) as a test runner.

**All of the code in this guide is available here:** [https://github.com/b-gran/react-test-renderer-demo](https://github.com/b-gran/react-test-renderer-demo)

## Test utilities included with React

React comes with several test utilities:
* [The full test renderer](https://reactjs.org/docs/test-renderer.html) which we'll be covering. Enables you to fully render your components and make assertions about the markup, all without using an actual DOM
* [The shallow test renderer](https://reactjs.org/docs/shallow-renderer.html) which enables you to make assertions about the _first level_ of your components' markup. An extremely fast renderer, but not capable of complex tests
* [The test utils](https://reactjs.org/docs/test-utils.html), a suite of helper utilities for making component assertions

Used together, these tools can produce robust component unit tests. They're light-weight, officially supported, and aren't authored by a third party.

This guide will explain how to use the **full test renderer (`react-test-renderer`)** and **the test utils (`react-dom/test-utils`)** to write component tests.

## Installing the tools

The **test utils** come with the [`react-dom`](https://www.npmjs.com/package/react-dom) package, so we need `react-dom` as a dependency
```bash
npm install react-dom
```


The **test renderer** is included in a separate package: [`react-test-renderer`](https://www.npmjs.com/package/react-test-renderer)
```bash
npm install -D react-test-renderer
```

## Using the test renderer

First, we need a component to test. We'll test a simple Checkbox component that renders a checkbox input & label.

```javascript
import React from 'react'
import PropTypes from 'prop-types'

export default class Checkbox extends React.Component {
  render () {
    return (
      <label>
        <input type='checkbox'
               checked={Boolean(this.props.checked)}
               onChange={this.props.onChange} />
        { this.props.text }
      </label>
    )
  }
}

Checkbox.propTypes = {
  text: PropTypes.node.isRequired,
  checked: PropTypes.bool,
  onChange: PropTypes.func.isRequired,
}
```

### `TestRenderer.create` - render a component tree
 `TestRenderer.create` renders the component into memory so we can make assertions about it. This API is the basis for all of the other tests we'll write.

`TestRenderer.create` returns a `TestRenderer` instance, which has methods for retrieving nodes in the rendered component tree.

The value returned by `TestRenderer.create` isn't very useful on its own.  In order to make assertions about the rendered markup, we'll need to use either `testRenderer.root` or `TestRenderer.getInstance()` to acquire a node we can make assertions on.

```javascript
// Render the component in memory
const tree = TestRenderer.create(
  <Checkbox text="a checkbox" onChange={() => {}} checked={true} />
)
```

### `testRenderer.root` - access the tree root
`testRenderer.root` returns the root of a component tree rendered by `TestRenderer.create`. `testRenderer.root` is a **"test instance"**, the type which actually contains methods for making assertions.

##### `testRenderer.root` vs `testRenderer.getInstance()` 
`testRenderer.root` will _always_ return a test instance, even if the root of the tree is a functional component or DOM node.

```javascript
// Just a stateless functional component
const SomeSFC = () => <p>hello world</p>

const tree = TestRenderer.create(
  <Checkbox text="a checkbox" onChange={() => {}} checked={true} />
)

const sfctree =  TestRenderer.create(<SomeSFC />)

const domNodeTree = TestRenderer.create(<div>foo</div>)

// .root will be a TestInstance for the SFC and the full-fledged Component
console.log(tree.root)
console.log(sfctree.root)
console.log(domNodeTree.root)

// For the Component, getInstance() returns the actual <Checkbox /> node
console.log(tree.getInstance())

// For the SFC and DOM node, getInstance() returns null
console.log(sfctree.getInstance())
console.log(domNodeTree.getInstance())
```

### `testInstance.find` - search the tree by predicate
`testInstance.find` traverses an entire component tree, applying a predicate to every node in the tree. If the predicate returns true for _any_ node in the tree, this assertion will pass. 

`testInstance.find` returns the _first_ node matching the predicate.

If the predicate does not match any node in the tree, the method will **throw** and the test will fail.

**Note:** `testInstance.find` will _not_ call the predicate on text nodes.

We will need to write our own simple helper to traverse text nodes.

**Helper for searching text nodes**
```javascript
// Returns a TestInstance#find() predicate that passes
// all test instance children (including text nodes) through
// the supplied predicate, and returns true if one of the
// children passes the predicate.
function findInChildren (predicate) {
  return testInstance => {
    const children = testInstance.children
    return Array.isArray(children)
      ? children.some(predicate)
      : predicate(children)
  }
}
```

**Searching the component tree for text**
```javascript
it('renders the supplied text somewhere in the tree', () => {
  const text = 'a checkbox'
  const tree = TestRenderer.create(
    <Checkbox text={text} onChange={() => {}} checked={true} />
  )

  // Verify that the checkbox text appears somewhere in the rendered tree.
  tree.root.find(findInChildren(node =>
    typeof node === 'string' &&
    node.toLowerCase() === text
  ))
});
```

`testInstance.find` returns a test instance with all of the same properties, including:
* `testInstance.props`: props of the node corresponding to the instance
* `testInstance.children`: children test instances of the node
* `testInstance.type`: type of the node (a string like `div` for DOM nodes, and the actual component class for composite elements)
* `testInstance.parent`: parent test instance of the nod3

### `testInstance.findByType` and `testInstance.findByProps` - search for specific types of nodes
Both `testInstance.findByType` and `testInstance.findByProps` search the component tree looking for nodes that match a property.

`testInstance.findByType` returns the first node with the specified type. The type can be a string (for DOM nodes) or a component class.

`testInstance.findByProps` returns the first node whose props contain the supplied props. The props are partially matched, so this assertion will pass if the node has additional props.

Both of these assertions **will fail if more than one node matches.**

To find _all_ nodes matching a specific property, see [`testInstance.findAllByType`](https://reactjs.org/docs/test-renderer.html#testinstancefindallbytype) and [`testInstance.findAllByProps`](https://reactjs.org/docs/test-renderer.html#testinstancefindallbyprops).

```javascript
it('renders exactly one checked <input />', () => {
  const tree = TestRenderer.create(
    <Checkbox text="a checkbox" onChange={() => {}} checked={true} />
  )

  // testInstance.findByType()
  //    Finds the first element in the tree with the specified type.
  //    You can pass in strings (corresponding to "primitive" React elements, like input and div)
  //    and you can also pass in custom component classes.
  //
  //    Throws (and fails the test) if there are no elements of the specified type or if
  //    there are more than instances.

  // Find the actual <input /> element
  const inputElementByType = tree.root.findByType('input')
  expect(inputElementByType.props.checked).toBe(true)
  expect(inputElementByType.props.type).toBe('checkbox')

  // We access the props of the element using testInstance.props

  // The <Checkbox /> itself
  const checkboxClassElement = tree.root.findByType(Checkbox)
  expect(checkboxClassElement.props.checked).toBe(true)
  expect(checkboxClassElement.props.text).toBe('a checkbox')

  // We can also use testInstance.findByProps() to assert that a single element in the tree
  // matches some props.

  // testInstance.findByProps() does partial matching of props, so we don't need to supply
  // every prop.
  const inputElementByProps = tree.root.findByProps({
    checked: true,
    type: 'checkbox',
  })

  // We have access to the actual element and can make further assertions
  expect(inputElementByProps.props.onChange).toBeInstanceOf(Function)
});
```

### Performing restricted searches of the tree
A common use case in component tests is searching for nodes within some other node's children. We can do these restricted searches by first `.find()`ing a node of interest, and then calling `.find()` on that node. 

```javascript
it('renders the <input /> within a <label />', () => {
  const tree = TestRenderer.create(
    <Checkbox text="a checkbox" onChange={() => {}} checked={true} />
  )

  // The testInstance.find() variants return test instances, so we can do a restricted
  // search of the <label /> to write a test that makes sure the label is clickable.
  const labelElement = tree.root.findByType('label')
  labelElement.findByProps({
    checked: true,
    type: 'checkbox',
  })
})
```

### Verifying that event handlers work
**Note: this method is extremely hacky but is provided for completeness. If you need to do these sorts of tests, consider using [enzyme](http://airbnb.io/enzyme/).**

We can use the `ReactTestUtils` to simulate DOM events. In order to simulate the elements, the component tree must be rendered in an actual DOM.

The basic method here is to first render a component into an actual DOM, find the node that will ultimately handle the DOM event, and then simulate the DOM event on that node.

The test utils will not propagate events, so we need to find the actual leaf node that handles the DOM event.

We will use `ReactTestUtils.findRenderedDOMComponentWithTag()` to find particular types of DOM elements in the rendered component tree.

```javascript
it('correctly responds to change events', () => {
  const checked = false
  const changeHandler = jest.fn()

  // Fully render the element as a detached node.
  // Requires a DOM. jest includes jsdom by default, but you'll need to set up jsdom
  // yourself if you aren't using jest.
  const element = ReactTestUtils.renderIntoDocument(
    <Checkbox text="a checkbox" onChange={changeHandler} checked={checked} />
  )

  // Find the actual <input /> element.
  // ReactTestUtils will not propagate events, so we need to find the exact element to
  // trigger the event on.
  const input = ReactTestUtils.findRenderedDOMComponentWithTag(element, 'input')


  // Simulate the change event on the input (what would happen if we clicked the checkbox
  // in the real DOM).
  ReactTestUtils.Simulate.change(input)

  // Verify that the change handler was called, and that it was passed an actual
  // <input /> element that has the correct checked prop.
  expect(changeHandler).toHaveBeenCalledTimes(1)
  const eventArgument = changeHandler.mock.calls[0][0]
  expect(eventArgument.target.checked).toBe(checked)
})
```

This test requires a DOM (actual or mocked). `jest` provides a DOM via [jsdom](https://github.com/jsdom/jsdom), so you will need to set one up yourself if you aren't using `jest` or running tests in the browser.
