![ReduRx: State Management with Observables](https://dl.dropboxusercontent.com/u/2179993/redurx-cropped.jpg)

## Basic Snippet
Always good to start out with a basic snippet:
```JavaScript
import { createState, createAction } from 'redurx';

const increment = createAction();
const decrement = createAction();
const state = createState({
  counter: 0
});
const counter = state('counter');

counter
  .asObservable()
  .subscribe(num => console.log(num));

state.connect();
// 0

counter
  .hookReducers(increment)
    .next(num => num + 1)
  .hookReducers(decrement)
    .next(num => num - 1);

increment();
// 1
increment();
// 2
decrement();
// 1
decrement();
// 0
```

## ReduRx: Like Redux but with Observables

[Redux](https://github.com/reactjs/redux), the predictable state container for JavaScript apps, has [three basic principles](https://github.com/reactjs/redux/blob/d4e57850e036dd5707c18400530ffc85138e0f8f/docs/introduction/ThreePrinciples.md). ReduRx also keeps those principles while using a tree of [RxJS](https://github.com/Reactive-Extensions/RxJS) Observables for managing state instead of arbitrary action objects and a single subscribe method. So let's look at ReduRx through the Redux lens:

### #1 Single Source of Truth

So this is also true with ReduRx. With Redux there is the store, and you describe the initial state of that store when it loads. You can do this either all at once when you create the store, or in parts as the reducer functions that manage the store are called.

ReduRx also maintains a single state tree; only each node in the tree has an observable associated with it. You can create the state tree all at once by calling `createState` with a value that you'd like to use as the initial state. When you're ready to use the state, you call `connect` on the state;

```javascript
import { createState } from 'redurx';

const state = createState({
  todos: {
    list: [],
    search: {
      filter: '',
      query: '',
      dirty: true
    },
    error: null
  },
  todonts: {
    list: [],
    search: {
      filter: '',
      query: '',
      dirty: true
    },
    error: null
  },
});

state.connect();
```
Each node is a function that can be used to access any child node by passing a dot delimited path to that node. So to get the node for the todo list from the state you could do:
```javascript
state('todos.list');
// or
state('todos')('list');
```
Each node has an observable associated with it that you can subscribe to. To get an observable call `asObservable` on the node.
```javascript
state('todos.list')
  .asObservable()
  .subscribe(list => {
    console.log(`The list: ${JSON.stringify(list)}`);
  });
// The List: []
```
You don't have to define your initial state at the outset however. You can figure out what the state for any part of your tree is at any point in the future. You can set the initial state by calling `setInitialState` on the node:
```javascript
import { createState } from 'redurx';

const state = createState();

state.setInitialState({
  todos: {
    list: [],
    search: {
      filter: '',
      query: '',
      dirty: true
    },
    error: null
  }
});

setTimeout(() => {
  state('todonts').setInitialState({
    list: [],
    search: {
      filter: '',
      query: '',
      dirty: true
    },
    error: null
  });

  state.connect();
}, 1000);
```
You can subscribe to changes on a node at any point though, even before the node has an initial value. If you've already connected the parent state before defining a new node, make sure to connect the child node after referencing it.
```javascript
import { createState } from 'redurx';

const state = createState();

state('todonts.list')
  .asObservable()
  .subscribe(list => {
    console.log(`The list: ${JSON.stringify(list)}`);
  });

state.connect();

setTimeout(() => {
  state('todonts').setInitialState({
    list: [],
    search: {
      filter: '',
      query: '',
      dirty: true
    },
    error: null
  }).connect();
}, 1000);

// ...time passes
// The List: []
```
### #2: State is Read only

Observables give you values, not the other way around.

### #3: Changes are made with Pure Functions

So if we're subscribing to changes in state, then state must be changeable. This is where ReduRx is like Redux, in that you can write reducer functions that take the previous value for a node, and some data, and return a new value for the node. You provide this additional data as a set of observables, and you provide your reducer functions through the node's `hookReducers` api:
```javascript
import Rx from 'rx';
// We defined and connected state somewhere else
import state from '../state';

const itemAction = new Rx.Subject();

const todoState = state('todos');
const listState = todoState('list');
const errorState = todoState('error');


const logStateWithType = (someState, type) => {
  someState
    .asObservable()
    .subscribe(list => {
      console.log(`The ${type}: ${JSON.stringify(list)}`);
    });
};

logStateWithType(listState, 'List')
// The List: []
logStateWithType(errorState, 'Error')
// The Error: null

todoState.hookReducers(itemAction)
  .next((state, item) => {
    return Object.assign({}, state, {
      list: [...state.list, item]
    });
  })
  .error((state, error) => {
    return Object.assign({}, state, {
      list: [],
      error: error.message
    });
  })

itemAction.onNext(42);
// The List: [42]
itemAction.onNext(50);
// The List: [42, 50]
itemAction.onError(new Error('AHHHH!'));
// The List: []
// The Error: 'AHHHH!'
```
Notice that the reducers are returning state for the parent node, but these changes are being propagated to the child nodes. The reverse is also true, in that if you hook reducers into child nodes it will update the state for the parent node. In this case, all updates to child nodes that result from an identical hooked observable will only cause a single update on all parent nodes.

To make these "action creator" observables easier to manage, ReduRx also includes a `createAction` function, that creates an update function with an associated observable. The create action function takes a callback that allows you to configure the action's observable, allowing you to transform the arguments to the action. Here's a somewhat complete example of what the state and business logic for a todo list app might look like:
```javascript
import axios from 'axios';
import { createAction } from 'redurx';
// We defined and connected state somewhere else
import state from '../state';

const todoState = state('todos').setInitialState({
  list: [],
  search: {
    filter: '',
    query: '',
    dirty: true
  },
  error: null
});

export const setTodoFilter = createAction((filters) => {
  return filters.distinctUntilChanged();
});

export const setTodoQuery = createAction((queries) => {
  return queries.distinctUntilChanged();
});

export const getTodos = createAction((submits) => {
  return submits
    .withLatestFrom(
      todoState('search.dirty').asObservable(),
      (_, dirty) => dirty
    )
    .filter(dirty => dirty)
    .withLatestFrom(
      todoState('search.filter').asObservable(),
      todoState('search.query').asObservable(),
      (_, filter, query) => ({ filter, query })
    )
    .flatMapLatest(params => axios.get('/api/todos', params))
    .map(result => result.data)
});

todoState('search.filter')
  .hookReducers(setTodoFilter)
    .next((state, filter) => filter);

todoState('search.query')
  .hookReducers(setTodoQuery)
    .next((state, query) => query);

todoState('search.dirty')
  .hookReducers(setTodoFilter, setTodoQuery)
    .next(() => true)
  .hookReducers(getTodos)
    .next(() => false)
    .error(() => false)

todoState('list')
  .hookReducers(getTodos)
    .next((state, list) => list)
    .error((state, err) => []);

todoState('error')
  .hookReducers(getTodos)
    .next(() => null)
    .error((_, err) => err);
```
Because you can hook reducers into any observable, you can even use other parts of the state to create computed properties. It's possible to create an infinite loop this way, so make sure that you don't hook state into its own children.
```javascript
import { createState } from 'redurx';

const state = createState({
  todos: {
    list: [{
      text: 'Some Todo',
      completed: false
    },{
      text: 'Some Other Todo',
      completed: true
    }],
    filteredList: [],
    filter: true
  }
});

state('todos.filteredList')
  .hookReducers(
    state('todos.list').asObservable(),
    state('todos.filter').asObservable()
  )
    .next((filtered, [list, filter]) => (
      list.filter(todo => todo.completed === filter)
    ));

// Call after you've hooked the state into itself
state.connect();

state('todos')
  .asObservable()
  .subscribe(todos => {
    console.log(`The Filtered List: ${JSON.stringify(todos.filteredList)}`)
  });
// The Filtered List: [{text:'Some Other Todo',completed:true}]
```

## One Way Data Flow

ReduRx maintains one way data flow through your application just like any Flux framework. Data is aggregated from various sources using observables, and given some input you'll get predictable output. [FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming) FTW! Here's how it works with ReduRx where the state is an object, with two values `bar` and `foo`. A single action creator is hooked into the base object, and we're subscribing to changes on the base object as well. The flow goes from green to blue to red:

![Diagram of one way data flow](https://dl.dropboxusercontent.com/u/2179993/redurx-data-flow.svg)


## How would I use this?
ReduRx, like Redux, can be used anywhere you'd like some functional state management. ReduRx is probably useful under a wider set of circumstances because you can subscribe to state changes anywhere in the tree, not just at the root. The obvious use for it is as a state container for React; and using something like [recompose](https://github.com/acdlite/recompose)'s [observable utilities](https://github.com/acdlite/recompose/blob/master/docs/API.md#observable-utilities) this turns out to be pretty simple:
```javascript
// See setting the observable config in the recompose docs
// You'll want to set it for RxJS 4
import { mapPropsStream } from 'recompose';

import state from '../state';
import {
  setTodoFilter,
  setTodoQuery,
  getTodos
} from '../actions/todos';
import TodoSearchBar from './todo-search-bar';

const enhance = mapPropsStream(propsStream => {
  return propsStream
    .combineLatest(
      state('todos').asObservable(),
      (props, { list, search }) => ({
        ...props,
        list,
        search
      })
    );
})

const TodoList = enhance(({ list, search }) => {
  const searchActions = { setTodoQuery, setTodoFilter, getTodos };
  return (
    <div>
      <TodoSearchBar {...search} {...searchActions} />
      <ul>
        {list.map(todo => <li>{todo}</li>)}
      </ul>
    </div>
  );
})
```

Pretty cool right! This project is still in it's early stages (read alpha), but that's how every project we couldn't live without got started. Some caveats: IE support right now is limited to Edge, and no Safari support on Windows; both because the code uses `WeakMap`. If a shim can be worked in the support would be expanded. The unit tests for the basic functionality are there, but this code has only had limited testing. Documentation, both in the code and standalone, need to be written. Bug reports and contributions are welcome. License as follows:

## License

ISC License

Copyright (c) 2016, Ryan Lynch

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
