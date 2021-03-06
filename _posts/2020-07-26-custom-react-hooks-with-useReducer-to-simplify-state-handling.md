---
layout: default
title: Custom hooks with useReducer to simplify state management
category: reactjs
tags:
- react
- reactjs
- redux
- hooks
- nodejs
- javascript
- state
---

Sometimes, storing even minimal state within React components can get unwieldy. For example, imagine a button component which allows a user to like an item in your application. Our component would need to allow toggling the state on or off, and should accept an initial value in case the user had previously favorited the item. In this post, we'll evaluate creating custom React hooks to encapsulate state management.

![Like button list]({{ site.url }}/assets/react_hooks_like_toggle.png)

For the use case I presented, the `useReducer` React hook seems like a simple solution.
```
const reducer = (state, action) => {
  switch (action.type) {
    case 'LIKE':
      return true;
    case 'UNLIKE':
      return false;
  }
}

const Button = (props) => {
  const [liked, dispatch] = useReducer(reducer, props.liked);

  return liked
    ? <Button  onClick={() => dispatch({ type: 'LIKE' })}>Unlike</Button>
    : <Button onClick={() => dispatch({ type: 'UNLIKE' })}>Like</Button>
};
```

Dispatching `LIKE` and `UNLIKE` events in response to click events would work, however, these actions probably require making an API call to persist the new state to the backend before updating the UI. We could add functions inside our component to first issue a request to the backend, and then dispatch an event to the reducer to update the state. While this sounds trivial, it quickly becomes a nightmare when you have to handle loading and error states for each API call. Our component would become significantly more complex and error prone if it had to manage async calls, internal state, and user interactions.

Instead, what we can do is encapsulate our reducer and API calls into a custom hook. Our hook will return an object containing loading, error, and like state as well as functions for liking and unliking. Let's start by implementing our reducer.

```
const useLikeToggle = ({ id, liked }) => {
  const actions = {
    LOADING: 'LOADING',
    LIKED: 'LIKED',
    UNLIKED: 'UNLIKED',
    ERROR: 'ERROR',
  };
  const reducer = (state, action) => {
    switch (action.type) {
      case 'LOADING':
        return { ...state, loading: true };
      case 'ERROR':
        return { ...state, loading: false, error: action.error };
      case 'LIKE':
        return { ...state, loading: false, liked: true };
      case 'UNLIKE':
        return { ...state, loading: false, liked: false };
    }
  }

  return [state];
};
```

All that's left is to define our toggle functions and expose them for our component to consume. I'll leave our api calls as simple promise returning functions in case you're using a RESTful or GraphQL backend.

```
const useLikeToggle = ({ id, liked }) => {
  const actions = {
    LOADING: 'LOADING',
    LIKED: 'LIKED',
    UNLIKED: 'UNLIKED',
    ERROR: 'ERROR',
  };
  const reducer = (state, action) => {
    switch (action.type) {
      case 'LOADING':
        return { ...state, loading: true };
      case 'ERROR':
        return { ...state, loading: false, error: action.error };
      case 'LIKE':
        return { ...state, loading: false, liked: true };
      case 'UNLIKE':
        return { ...state, loading: false, liked: false };
    }
  }
  const [state, dispatch] = useReducer(reducer, { loading: false, liked });

  const like = () => {
    dispatch({ type: actions.LOADING });
    api.like(id)
      .then(() => dispatch({ type: actions.LIKED }))
      .catch(error => dispatch({ type: actions.ERROR }, error));
  };

  const unlike = () => {
    dispatch({ type: actions.LOADING });
    api.unlike(id)
      .then(() => dispatch({ type: actions.UNLIKED }))
      .catch(error => dispatch({ type: actions.ERROR }, error));
  };

  return [state, { like, unlike }];
};
```

Now, our button component can make use of our new hook.

```
const Button = (props) => {
  const [{ loading, error, liked }, { like, unlike }] = useLikeToggle({ id: props.id, liked: props.liked });

  return liked
    ? <Button disabled={loading} onClick={unlike}>Unlike</Button>
    : <Button disabled={loading} onClick={like}>Like</Button>
};
```

In this post, we've explored how an interaction as simple as toggling a boolean value can sometimes require complex state management. By creating custom hooks utilizing `useReducer`, we significantly simplified our code and made it easy to share API logic in other components.
