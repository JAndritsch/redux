# Optimization

[WIP: Intro here]

## Connect components to state at the lowest level possible

One common mistake beginner Redux users may run into is connecting their highest level components to state. At first this may seem tempting to create an `App` component that subscribes to your state so that when state changes, your entire application will automatically update. The problem with this approach, however, is that any action dispatched will cause your *entire* app to re-render, regardless if it was interested in the piece of state that changed or not. Doing this could result in major performance degredation in large Redux applications.

Instead of connecting your state to parent components that are farther up the tree, you should connect your lowest level components to state so that only they update when state changes.

To help illustrate this, let's pretend our application has the following components:

One that displays data from state:

```js
class Child1 extends React.Component {
  render() {
    return (
      <div>{this.props.someState}</div>
    );
  }
}

export default Child1;
```

One that displays static text only:

```js
class Child2 extends React.Component {
  render() {
    return {
      <div>Some static text</div>
    };
  }
}

export default Child2;
```

And a container that wraps the two components:

```js
class App extends React.Component {
  render() {
    return (
      <div>
        <Child1 someState={this.props.someState}/>
        <Child2 />
      </div>
    );
  }
}

const mapStateToProps = (state) => {
  return {
    someState: state.someState
  };
};

export default connect(mapStateToProps)(App);
```

In this example, a connection to the Redux state is maintained at the top-level `App` component.  This means that any time `someState` changes, the `App` component and all of its children will be rendered.

Since the `Child1` component is the only component that's interested in the `someState` prop, it would be wasteful to have the connection at the `App` level because we would be re-rendering the `App` and `Child2` components as well. To prevent the `App` and other children that aren't interested in `someState` from extraneous renders, we can move the subscription to the Redux store a level down into the component that needs it. In this case, that would be `Child1`.

Example:

Move the connection to the lowest component that needs it:

```js
class Child1 extends React.Component {
  render() {
    return (
      <div>{this.props.someState}</div>
    );
  }
}

const mapStateToProps = (state) => {
  return {
    someState: state.someState
  };
};

export default connect(mapStateToProps)(Child1);
```

Disconnect our `App` component to prevent it and all of its children from needless renders:

```js
class App extends React.Component {
  render() {
    return (
      <div>
        <Child1 someState={this.props.someState}/>
        <Child2 />
      </div>
    );
  }
}

export default App;
```

Now when `someState` changes, only the `Child1` component will re-render.

Although this approach may result in having many more components connected to state, it's still better performance-wise than having one connection at the highest level of your application.

## Redefine `shouldComponentUpdate` when needed

By default, components connected to state via React Redux's `connect` method are automatically given an implementation for `shouldComponentUpdate`. I won't go into implementation details, but the gist of it is that any changes to the state your component connects to will result in `shouldComponentUpdate` returning `true`. While this implementation may work for some components, there may be times where that default behavior may not be fast enough for your application.

One example of where this may apply is in singleton components that are used to display information for several objects. By "singleton" components, I'm referring to a connected component that's instantiated one time and is reused by several objects in state.

Consider the following state:

```js
{
  currentItemIndex: 0,
  items: [
    {
      id: 1,
      color: 'blue',
      size: 'M'
    },
    {
      id: 2
      color: 'blue',
      size: 'M'
    },
    {
      id: 3
      color: 'red',
      size: 'S'
    }
  ]
}
```

The component:

```js
class CurrentItem extends React.Component {
  render() {
    return (
      <div>Color: {this.props.currentItem.color}</div>
      <div>Size: {this.props.currentItem.size}</div>
    );
  }
}

const mapStateToProps = (state) => {
  return {
    currentItem: state.items[state.currentItemIndex]
  };
};

export default connect(mapStateToProps)(CurrentItem);
```

And our reducer for the `currentItemIndex` state:

```js
export default (state = 0, action) => {
  switch (action.type) {
    case 'SHOW_NEXT_ITEM':
      return state + 1;
    case 'SHOW_PREVIOUS_ITEM':
      return state - 1;
    default:
      return state;
  }
};
```

In this example, we have a list of `items` and a `currentItemIndex` in our state. Our `CurrentItem` component is used to represent the active or current item being looked at (only one item is visible at a time). Let's pretend that the `currentItemIndex` state can change rapidly, say via keystroke. Whenever the user presses the left arrow key, an action fires that decrements the `currentItemIndex`. When the right arrow key is pressed, the `currentItemIndex` gets incremented.

Our connected component does not define `shouldComponentUpdate`. It instead relies on the default one provided by React Redux. Consider what would happen if we were viewing the first item in the list and then pressed the right arrow key to dispatch the following action:

```js
{
  type: 'SHOW_NEXT_ITEM'
}
```

When the reducer runs, it will get the current index (which is 0), then return that number incremented by 1. React Redux would recognize that the `currentItemIndex` state has changed and pass the new value to the connected component for a re-render.  But if we look at what information our component consumes, we can see that both the previous item and the next item have the exact same attributes (with the exception of the `id` prop). 


The item we started at:

```js
{
  id: 1,
  color: 'blue',
  size: 'M'
}
```

The item we moved to:

```js
{
  id: 2,
  color: 'blue',
  size: 'M'
}
```

In this case, we'd end up wasting time rendering because the color and size properties haven't changed between the two items being viewed.  Instead of relying on the default implementation of `shouldComponentUpdate`, we can implement a smarter version that knows not to re-render itself if all of the display data is the same.

Here is an example:

```js
class CurrentItem extends React.Component {

  shouldComponentUpdate(nextProps) {
    let colorHasChanged = this.props.currentItem.color !== nextProps.currentItem.color;
    let sizeHasChanged = this.props.currentItem.size !== nextProps.currentItem.size;

    return colorHasChanged || sizeHasChanged;
  }

  ...
```


With that logic in place, our component will only re-render itself if the `color` or `size` attributes have changed. This means that we can continue cycling between item 0 and 1 and not have our component waste time rendering over and over.

## Use `initialProps` when possible

You are probably familiar with `mapStateToProps` when connecting a component to your Redux state. In a simple todo application, you may have a component connected to state via the following code:

```js
class TodoItem extends React.Component {}

const mapStateToProps = (state, ownProps) => {
  const { todos } = state;
  const { id } = ownProps;

  const todo = todos.byId[id];

  return {
    todo  
  };
};

export default connect(mapStateToProps)(TodoItem);

// Ex usage: <TodoItem id="123" />;
```

In this example, our `mapStateToProps` serves as a means to obtain the specific `TodoItem` from our `todos` state via the `id` property. As you can see, the `id` property is static and will never change.

Instead of defining `mapStateToProps` as a function that returns an object, you can define a factory function that returns your `mapStateToProps` function. Here's what that might look like:

```js
const makeMapStateToProps = (initialState, initialOwnProps) => {
  const { id } = initialOwnProps
  const mapStateToProps = (state) => {
    const { todos } = state
    const todo = todos.byId[id]
    return {
      todo
    }
  }
  return mapStateToProps;
}

export default connect(makeMapStateToProps)(TodoItem);
```

The reason you may consider doing this is due to the performance implications of calculating props in your `mapStateToProps` function. Since the `id` property of the `TodoItem` is static, we can expect that it won't ever need to be recalculated once set.

## Use memoization for computation heavy functions

In Redux applications, the majority of computation will probably take place in your reducers. As we know, reducers are meant to accept a current state, then translate an action into the next piece of state. Often times, these computations are simple and happen very quickly.

Here's an example showing a reducer for a collection of items:

```js
// reducers/filteredItems.js
const allItems = []; // The complete list of items (assume this never changes)

export default (state = [], action) => {
  switch (action.type) {
    case 'FILTER_BY_COLOR':
      return allItems.filter(item => item.color === action.color);
    default:
      return state;
  }
};
```

This reducer handles the action type `FILTER_BY_COLOR` by taking the color provided from the action and returning a new array representing the filtered items. Consider what would happen when the following action dispatches:

```js
{
  type: 'FILTER_BY_COLOR',
  color: 'Blue'
}
```

When the reducer executes, it will iterate over `allItems` and select only the ones that are blue. But what happens if that action fires again, or perhaps even multiple times in succession, executing that same loop multiple times? While this may not seem like a big deal at first, consider what happens if our list contains 50,000 items. We'd effectively be iterating over that collection and running the same calculation over and over again. Wouldn't it be much more efficient if our code could remember that it already performed this calculation before?

That's where memoization helps. Memoization is a technique that allows your code to avoid recomputation of outputs when given previously supplied inputs. 

Here's a rudimentary example of adding memoization to our reducer:

```js
const allItems = [];

const previousResults = {}; // a cache to hold the previous calculations

export default (state = [], action) => {
  switch (action.type) {
    case 'FILTER_BY_COLOR':
      let color = { action };
      if (previousResults[color]) {
        // We've already done this calculation once, so just return a copy of the previous result.
        return previousResults[color].slice(0); 
      } else {
        // First time. Do the calculation once and then save the results.
        let filtered = allItems.filter(item => item.color === action.color);
        previousResults[color] = filtered;
        return filtered;
      }
    default:
      return state;
  }
};
```

With those changes in place, our filtering logic will only fire once and subsequent calls to filter for blue items will use the results from the first calculation.

If you're interested in learning better ways to introduce memoization in your Redux application then check out [Reselect](https://github.com/reactjs/reselect).

## Normalize your data

WIP

