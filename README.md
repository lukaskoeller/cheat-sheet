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

### Context

1. Create Context
```ts
const FormContext = createContext<Context>({} as Context);
```

2. Add reducer
```ts
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
```ts
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
```ts
export const useFormState = () => useContext(FormContext);
```

4. Implement Provider
```ts
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

```ts
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
```ts
export const SelectCountryFormComponent = () => {
  const { onCountryChange } = useFormAPI();

  return <SelectCountry onChange={onCountryChange} />;
};
```
