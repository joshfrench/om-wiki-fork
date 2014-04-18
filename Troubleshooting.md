### Why won't my component unmount?

### Async rendering side effects
Om batches updates and executes them asynchronously on requestAnimationFrame. One consequence of that is that updates will not happen on the same cycle as the state change that triggered them. This may cause certain API that expect that (e.g. event.preventDefault()) to fail. You should be able to handle such edge cases by finding a workaround within Om, or alternatively reaching into React.