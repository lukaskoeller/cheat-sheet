# cheat-sheet

## React

### `useMemo` and `useCallback`
* **Wth memoization**: between re-renders - cache it during the initial render, and return the reference to that saved value during consecutive renders.
* **Without memoization**: non-primitive values (arrays, object, function) - re-created on every render.
* **Disadvantage**: On inital render, they slow down the app
* **Re-Renders**: When props change, state changes, parent component re-renders.

* `useMemo` are good for non-primitive values; avoid expensive calculations
  * Remove for pure javascript operations except for calculating factorials of big numbers.
  * Ideally combine with returning JSX of e.g. a list.
* `useCallback` is best for functions.
* `React.memo` for components

* Only memoize if: every single prop and the component itself are memoized.
