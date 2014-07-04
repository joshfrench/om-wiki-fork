### Why won't my component unmount?

### Async rendering side effects
Om batches updates and executes them asynchronously on requestAnimationFrame. One consequence of that is that updates will not happen on the same cycle as the state change that triggered them. This may cause certain API that expect that (e.g. event.preventDefault()) to fail. You should be able to handle such edge cases by finding a workaround within Om, or alternatively reaching into React.

### Why do events lose all data when put on a core.async channel or logged to the console?

### React's SyntheticEvents 
React's [SyntheticEvent documentation](facebook.github.io/react/docs/events.html) describes SyntheticEvents as "a cross-browser wrapper around the browser's native event". The implementation of SyntheticEvent uses the object pool pattern to reduce the frequency of garbage collection. When an event is fired in React, an object from the event pool is used to represent the event. After each event loop, the objects backing these events are cleared and released back to the React's object pool to be used again. A SyntheticEvent can be made to persist across event loops by calling persist on the SyntheticEvent object(e.g. event.persist()).
