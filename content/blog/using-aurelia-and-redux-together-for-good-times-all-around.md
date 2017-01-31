+++
date = "2016-03-17T12:46:52+11:00"
title = "Using Aurelia and Redux together for good times all around"
draft = false
+++

<p>Since playing with <a href="https://facebook.github.io/react/">React</a> and <a href="https://redux.js.org/">Redux</a> I've really missed the separation of data flow and view logic this affords when working with <a href="https://aurelia.io/">Aurelia</a> - in my opinion a fantastic framework to work with, but lacks data management features out-of-the-box. But alas, Redux to the rescue!</p>

<p>I'm going to show you how to use Redux with Aurelia in a pretty simple example with config for <a href="https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd?hl=en">Redux Chrome DevTools</a>, <a href="https://github.com/fcomb/redux-logger">redux-logger</a> and <a href="https://github.com/gaearon/redux-thunk">redux-thunk</a>. Basically we're going to create a store and DI it everywhere, just like how in a React app you'd inject it through the router or create some kind of global. This assumes basic familiarity with Redux and Aurelia and the code can be found <a href="">here</a>.</p>

<p>First, let's install all the bits we need:  </p>

<pre><code>npm install --save redux redux-logger redux-thunk
</code></pre>

<p>FWIW I am using: <br />
aurelia-* beta 1 <br />
redux 3.3.1 <br />
redux-logger 2.6.1 <br />
redux-thunk 2.0.1</p>

<p>Nice, now let's make a new file called <code>store.js</code> and put all our Redux code there. Normally I like to make a folder for all of this and break it up, but for the purpose of this example we'll keep it together.</p>

<pre>
<code data-language="js">import { createStore, applyMiddleware, compose } from 'redux'
import thunkMiddleware from 'redux-thunk'
import createLogger from 'redux-logger'

const loggerMiddleware = createLogger();

const sessionReducer = (state = {
  error: null,
  loggedIn: false
}, action) => {
  switch (action.type) {
    case 'LOGIN_SUCCESS':
      return {
        ...state,
        loggedIn: true,
        error: null
      };
    case 'LOGIN_FAILURE':
      return {
        ...state,
        loggedIn: false,
        error: action.error
      };
    default:
      return state;
  }
};

function configureStore() {
  return createStore(
    sessionReducer, // or combine some reducers
    compose(
      applyMiddleware(
        thunkMiddleware,
        loggerMiddleware
      ),
      window.devToolsExtension ? window.devToolsExtension() : undefined
    )
  )
}

export const reduxStore = configureStore();
</code></pre>

Now let's go over to our `app.js` (or whatever your root ViewModel is) and DI it in:



<pre>
<code data-language="js">import {inject} from 'aurelia-framework';
import {reduxStore} from 'store';

@inject(reduxStore)
export class App {
  constructor(store) {
    this.store = store;
    this.store.dispatch({type: 'LOGIN_FAILURE', error: 'Oops! Wrong password matey.'});
  }
}
</pre>

<p></code></p>

<p>Because we're Dependency Injecting the store with <code>@inject</code>, we can do this in many different ViewModels and know we're getting the same instance everywhere, meaning we can just <code>store.subscribe()</code> and <code>store.dispatcher.dispatch()</code> in many places and know everything is talking to the same <code>store</code>.</p>

<p>Let me know if you found this easy to follow or have any questions, I've only recently started writing these kinds of quick tip blog posts!</p>
