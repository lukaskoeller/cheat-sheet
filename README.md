# Naming conventions
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

## Layers
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

# JavaScript

## Event Delegation & Bubbling
Source: https://programmingwithmosh.com/javascript/javascript-event-bubbling-and-event-delegation/

* When **users interact** with the page, their actions are captured in form of **events**.
* In order to do something in response to an event, you need to register a listener

### 

# React

## `useMemo` and `useCallback`
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

## Context
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

## Data Fetching

### Simple Fetch
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

### Data providers to abstract away fetching
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

## Higher-Order Components (HOC)
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

### Use Case #1: Enhancing callbacks and React lifecycle events

> \[...] encapsulate the logic of “something triggered onClick callback - send some logging events” somewhere, and then just re-used it in any component I want, without changing the code of those components in any way.

Create a `withLoggingOnClick` function, that:

* accepts a component as an argument
* intercepts its onClick callback
* sends the data that I need to the whatever external framework is used for logging
* returns the component with onClick callback intact for further use

Use Cases:
* to enhance callbacks and React lifecycle events with additional functionality, like sending logging or analytics events
* to intercept DOM events, like blocking global keyboard shortcuts when a modal dialog is open
* to extract a piece of Context without causing unnecessary re-renders in the component

```tsx
type Base = { onClick: () => void };
export const withLoggingOnClickWithProps = <TProps extends Base>(Component: ComponentType<TProps>) => {
  // our returned component will now have additional logText prop
  return (props: TProps & { logText: string }) => {
    
    useEffect(() => {
      console.log('log on mount');
    }, []);
  
    const onClick = () => {
      // accessing it here, as any other props
      console.log('Log on click: ', props.logText);
      props.onClick();
    };

    return <Component {...props} onClick={onClick} />;
  };
};
```

Usage
```tsx
const Page = () => {
  return (
    <ButtonWithLoggingOnClickWithProps onClick={onClickCallback} logText="this is Page button">
      Click me
    </ButtonWithLoggingOnClickWithProps>
  );
};
```

### Use Case #2: Intercepting DOM events

> \[...] implement some sort of keyboard shortcuts functionality on your page. When specific keys are pressed, you want to do various things, like open dialogs, creating issues, etc.

```tsx
export const withSupressKeyPress = <TProps extends unknown>(Component: ComponentType<TProps>) => {
  return (props: TProps) => {
    const onKeyPress = (event) => {
      event.stopPropagation();
    };

    return (
      <div onKeyPress={onKeyPress}>
        <Component {...props} />
      </div>
    );
  };
};
```

```tsx
const ModalWithSupressedKeyPress = withSupressKeyPress(Modal);
const DropdownWithSupressedKeyPress = withSupressKeyPress(Dropdown);
// etc
```

### Use Case #3: Context selectors

> When we have a Context, but we want to memoize parts of the value the context provides, we can use HOC to momoize the component that uses this part of 
value of that context.

```tsx
export const withFormIdSelector = <TProps extends unknown>(
  Component: ComponentType<TProps & { formId: string }>
) => {
  const MemoisedComponent = React.memo(Component) as ComponentType<
    TProps & { formId: string }
  >;

  return (props: TProps) => {
    const { id } = useFormContext();

    return <MemoisedComponent {...props} formId={id} />;
  };
};
```
Sandbox: https://codesandbox.io/s/hocs-context-lwudbb?file=/src/page.tsx

#### Generic React context selector

```tsx
export const withContextSelector = <TProps extends unknown, TValue extends unknown>(
  Component: ComponentType<TProps & Record<string, TValue>>,
  selectors: Record<string, (data: Context) => TValue>,
): ComponentType<Record<string, TValue>> => {
  // memoising component generally for every prop
  const MemoisedComponent = React.memo(Component) as ComponentType<Record<string, TValue>>;

  return (props: TProps & Record<string, TValue>) => {
    // extracting everything from context
    const data = useFormContext();

    // mapping keys that are coming from "selectors" argument
    // to data from context
    const contextProps = Object.keys(selectors).reduce((acc, key) => {
      acc[key] = selectors[key](data);

      return acc;
    }, {});

    // spreading all props to the memoised component
    return <MemoisedComponent {...props} {...contextProps} />;
  };
};
```
