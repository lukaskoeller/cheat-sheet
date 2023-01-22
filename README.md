# cheat-sheet

## Naming conventions
Source: https://www.developerway.com/posts/react-project-structure

Use npm/pnpm workspaces and split up in packages.

```
/my-feature-name
  /assets     // if I have some images, then they go into their own folder
    logo.svg
  index.tsx   // main feature code
  test.tsx    // tests for the feature if needed
  stories.tsx // stories for storybooks if I use them
  styles.(tsx|scss) // I like to separate styles from component's logic
  types.ts    // if types are shared between different files within the feature
  utils.ts    // very simple utils that are used *only* in this feature
  hooks.tsx   // small hooks that I use *only* in this feature
```

```
/my-feature-name
  ... // index the same as before
  /header
    index.tsx
    ... // etc, exactly the same naming here
  /footer
    index.tsx
    ... // etc, exactly the same naming here
```

Example: https://github.com/developerway/example-react-project

### Layers
* “data” layer - queries, mutation and other things that are responsible for connecting to the external data sources and transforming it. Used only by UI layer, doesn’t depend on any other layers.
* “shared” layer - various utils, functions, hooks, mini-components, types and constants that are used across the entire package by all other layers. Doesn’t depend on any other layers.
* “ui” layer - the actual feature implementation. Depends on “data” and “shared” layers, no-one depends on it
* (“state”) layer - if external state management library is used. Bridge between “data” and “ui”. Is using “shared” and “data”, while “ui” is using “state”.

We can structure a layer in a hierarchical way. The rules are:

* only main files (i.e. “index.ts”) in a folder can have sub-components (sub-modules) and can import them
* you can import only from the “children”, not from “neighbours”
* you can not skip a level and can only import from direct children

```
/my-feature-package
  /shared
  /ui
  /data
  index.ts
  package.json
```

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

### Context
> Reference: https://www.developerway.com/posts/how-to-write-performant-react-apps-with-context

1. Create Context
```tsx
const FormContext = createContext<Context>({} as Context);
```

2. Add reducer
```tsx
type Actions =
  | { type: 'updateName'; name: string }
  | { type: 'updateCountry'; country: Country }
  | { type: 'updateDiscount'; discount: number };

const reducer = (state: State, action: Actions): State => {
  switch (action.type) {
    case 'updateName':
      return { ...state, name: action.name };
    case 'updateDiscount':
      return { ...state, discount: action.discount };
    case 'updateCountry':
      return { ...state, country: action.country };
  }
};
```

3. Create Provider
```tsx
export const FormDataProvider = ({ children }: { children: ReactNode }) => {
  const [state, dispatch] = useReducer(reducer, {} as State);

  const api = useMemo(() => {
    const onSave = () => {
      // send the request to the backend here
    };

    const onDiscountChange = (discount: number) => {
      dispatch({ type: 'updateDiscount', discount });
    };

    const onNameChange = (name: string) => {
      dispatch({ type: 'updateName', name });
    };

    const onCountryChange = (country: Country) => {
      dispatch({ type: 'updateCountry', country });
    };

    return { onSave, onDiscountChange, onNameChange, onCountryChange };
    // no more dependency on state! The api value will stay the same
  }, []);

  return <FormContext.Provider value={{ state, ...api }>{children}</FormContext.Provider>;
};
```

3. Create hook with Context
```tsx
export const useFormState = () => useContext(FormContext);
```

4. Implement Provider
```tsx
export default function App() {
  return (
    <FormDataProvider>
      <Form />
    </FormDataProvider>
  );
}
```

5. Split into two `Context`s to improve performance
If parts of the app only uses the api part or only the state part we can split up the Context into two, to avoid unneccessary re-rendering.

```tsx
type State = {
  name: string;
  country: Country;
  discount: number;
};

type API = {
  onNameChange: (name: string) => void;
  onCountryChange: (name: Country) => void;
  onDiscountChange: (price: number) => void;
  onSave: () => void;
};

const FormDataContext = createContext<State>({} as State);
const FormAPIContext = createContext<API>({} as API);

const FormDataProvider = () => {
  const [state, dispatch] = useReducer(reducer, {} as State);

  const api = useMemo(() => {
    // ... all `on*` callbacks

    return { onSave, onDiscountChange, onNameChange, onCountryChange };
  }, []);

  return (
    <FormAPIContext.Provider value={api}>
      <FormDataContext.Provider value={state}>{children}</FormDataContext.Provider>
    </FormAPIContext.Provider>
  );
};
```

```ts
export const useFormData = () => useContext(FormDataContext);
export const useFormAPI = () => useContext(FormAPIContext);
```

Consume like this:
```tsx
export const SelectCountryFormComponent = () => {
  const { onCountryChange } = useFormAPI();

  return <SelectCountry onChange={onCountryChange} />;
};
```

### Data Fetching

#### Simple Fetch
```tsx
const Component = () => {
  const [data, setData] = useState();

  useEffect(() => {
    // fetch data
    const dataFetch = async () => {
      const data = await (
        await fetch(
          "https://run.mocky.io/v3/b3bcb9d2-d8e9-43c5-bfb7-0062c85be6f9"
        )
      ).json();

      // set state when the data received
      setState(data);
    };

    dataFetch();
  }, []);

  return <>...</>
}
```

#### Data providers to abstract away fetching
Ability to fetch data in one place of the app and access that data in another, bypassing all components in between. Essentially like a mini-caching layer per request.

```tsx
const Context = React.createContext();

export const CommentsDataProvider = ({ children }) => {
  const [comments, setComments] = useState();

  useEffect(async () => {
    fetch('/get-comments').then(data => data.json()).then(data => setComments(data));
  }, [])

  return (
    <Context.Provider value={comments}>
      {children}
    </Context.Provider>
  )
}

export const useComments = () => useContext(Context);
```

### Higher-Order Components (HOC)
Source: https://www.developerway.com/posts/higher-order-components-in-react-hooks-era

> Advanced technique to re-use components logic that is used for cross-cutting concerns.

* The key here is the return part of the function - it’s **just a component**, like any other component.

Simple Example:
```tsx
// accept a Component as an argument
const withSomeLogic = (Component) => {
  // do something

  // return a component that renders the component from the argument
  return (props) => <Component {...props} />;
};
```

```tsx
const Button = ({ onClick }) => <button onClick={func}>Button</button>;
const ButtonWithSomeLogic = withSomeLogic(Button);
```

#### Use Case #1: Enhancing callbacks and React lifecycle events

> \[...] encapsulate the logic of “something triggered onClick callback - send some logging events” somewhere, and then just re-used it in any component I want, without changing the code of those components in any way.

Create a `withLoggingOnClick` function, that:

* accepts a component as an argument
* intercepts its onClick callback
* sends the data that I need to the whatever external framework is used for logging
* returns the component with onClick callback intact for further use

```tsx
type Base = { onClick: () => void };

// just a function that accepts Component as an argument
export const withLoggingOnClick = <TProps extends Base>(Component: ComponentType<TProps>) => {
  return (props: TProps) => {
    const onClick = () => {
      console.log('Log on click something');
      // don't forget to call onClick that is coming from props!
      // we're overriding it below
      props.onClick();
    };

    // return original component with all the props
    // and overriding onClick with our own callback
    return <Component {...props} onClick={onClick} />;
  };
};

Usage
```tsx
export const ButtonWithLoggingOnClick = withLoggingOnClick(SimpleButton);
```
```
