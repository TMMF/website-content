# React Re-Mounting vs. Re-Rendering

What would the following lines of code do when React is rendering a component? Would they cause `Counter` to get re-mounted or re-rendered?

```tsx
// 'name' is a variable that is either "A" or "B"

// Passing in the name as a prop
<Counter name={name} />

// Ternary expression with two written cases of Counter
{name === "A" ? <Counter name="A" /> : <Counter name="B" />}

// Ternary expression with a Counter and a different element
{name === "A" ? <Counter name="A" /> : <p>EMPTY</p>}
```

If you said that the first two will re-render `Counter` while the third will cause a re-mount, then you are correct! You can verify this for yourself with this [codesandbox link](https://codesandbox.io/s/cocky-water-7rz6g?file=/src/App.js). The "Basic" section shows all three cases mentioned above.

## The Basic Case

To provide some context on `Counter`, it's a simple component that holds an internal count (with the `useState` hook) for the number of times it has been pressed:

```tsx
const Counter = (props) => {
  const [count, setCount] = useState(0)
  const increment = () => setCount(count + 1)

  return (
    <>
      <button onClick={increment}>{props.name}</button>
      <p>{count}</p>
    </>
  )
}
```

From this component, the most basic use case would simply be passing in the name as a prop as follows:

```tsx
// Passing in the name as a prop
<Counter name={name} />
```

This is probably the most common and intuitive case. When React receives new props for a component, it will re-render the component. This results in any internal `useState` hooks maintaining their internal data – which in our case means the count remains the same.

## The Unintuitive Re-Render Case

The next case is less intuitive:

```tsx
// Ternary expression with two written cases of Counter
{name === "A" ? <Counter name="A" /> : <Counter name="B" />}
```

At first glance, there appears to be two separate components that are being used in order to render counters; each counter associated with a different name. This could lead one to believe that both counters will go through a mount and unmount process when switching between them. However, that is not the case.

Since these are both the same component type, React actually sees this as identical to the first case. Under the hood, React uses a Virtual DOM reconciler based on a *[Fiber Architecture](https://giamir.com/what-is-react-fiber)* that determines how to update components (re-rendering, mounting, unmounting, etc). This reconciler uses the type of the component and the props in order to determine what lifecycle operations to take. In this case, both branches of the ternary use the same component type, but different props. This causes the reconciler to re-render the component and simply change the props passed in.

### Why is this important?

> *"Of course this case is the same! The code is functionally equivalent! Why are you telling me this?"*
- *someone who's too impatient to continue reading*

Consider an application with tabs. You may have the same components that stay within the same locations across tabs. Since the components line up within the Virtual DOM hierarchy between tab transitions, this can unexpectedly cause the same re-rendering behavior to occur.

## The Intuitive Re-Mount Case

```tsx
// Ternary expression with a Counter and a different element
{name === "A" ? <Counter name="A" /> : <p>EMPTY</p>}
```

Alright, back to the intuitive. To tie it all together, the reason why this case re-mounts is quite simply due to the change in component types. On the left branch we have a `Counter` component while on the right branch we have a `p` element. As mentioned above, React's reconciler uses these component types in order to determine what operations to take. Since the types are different when you switch branches, it will unmount the component that was mounted and mount the component that was unmounted.

This unmounting process throws away any data saved within the component's state. Likewise, the mounting process causes component state to initialize with default values (e.g. the initial value passed into a `useState` hook). This is what causes our count state to reset to `0` whenever switching between branches.

## What do I do with this information?

Well, there are a few real world cases where you may want to specifically have re-rendering or re-mounting behavior. Let's continue to use the `Counter` component and build upon it.

### Replicating Re-Mounting

Let's say that we have a web app that allows you to manage multiple users. Each of these users has a `Counter` component and allows you to save their respective counts. You may write the user component like:

```tsx
const User = (props) => {
	...
  return (
    <>
      <Counter name={props.name} />
			...
    </>
  )
}
```

And with this `User` component, you set up a tabs component that shows one user at a time.

The problem that will occur here is that the `Counter` component's state won't reset between users. This means that when you switch between the tabs, the count will stay the same and you may accidentally save the wrong count for a given user. Extrapolating this out from a simple counter, your app may cause you to save sensitive data to the wrong user – which is a severe breach of security.

So, "how do I fix this?"

Well, the solution is a `useEffect` hook. We want to listen for changes to props within the `Counter` component in order to reset the state manually ourselves:

```tsx
const Counter = (props) => {
  const [count, setCount] = useState(0)
  const increment = () => setCount(count + 1)
  
	useEffect(() => {
		setCount(0)
	}, [props.name])

  ...
```

All that we've added here is a simple `useEffect` hook that runs every time the `name` prop changes for the component. This causes the internal `count` to get reset and our 'sensitive' data to avoid leaking into other users.

You can confirm this for yourself by heading to the [same codesandbox link](https://codesandbox.io/s/cocky-water-7rz6g) as before and checking out the "Replicating Re-Mounting" section. Although it is defined the exact same way as the first case from the "Basic" section, it acts most similarly to the third case with its re-mounting.

### Replicating Re-Rendering

Ok, now let's take the original `Counter` component in a different route. This time, let's assume that we have a `Counter` that only exists on one tab out of many. We may want to replicate the re-rendering functionality in order to save data when you switch back-and-forth between tabs. That way, as a user, you can work in multiple tabs without losing any data.

What I've described is basically caching the data outside of the component's state in order to prevent it from resetting. You can approach this with a variety of methods: from Redux, to React Context, to a simple cache object external from the component.

For our example, we'll do a simple cache just to show the basics. To start, we want to define a cache for us to use and a way for us to update that cache:

```tsx
const cache = {}
const Counter = (props) => {
	const [count, setCount] = useState(cache[props.name] ?? 0)
	const increment = () => setCount(count + 1)
  ...
```

Now we want a way to update the cache when the component’s `name` prop changes (so that we cache data for each user):

```tsx
const cache = {}
const Counter = (props) => {
	const [count, setCount] = useState(cache[props.name] ?? 0)
	const increment = () => setCount(count + 1)

	useEffect(() => {
    setCount(cache[props.name] ?? 0)

    return () => {
      cache[props.name] = count
    };
  }, [props.name])

	...
```

This `useEffect` will also run during mount and likewise the cleanup function will run during unmount. 

But wait! This code has a problem. When the cleanup function is created, `count` is captured within a [closure](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) and it’ll save the wrong data into the cache. If we try to fix this by adding `count` as a dependency for the hook, then it’ll cause the page to crash due to a circular reference.

To solve this issue, we can use the `useRef` hook in order to use its mutative `current` field:

```tsx
const cache = {}
const Counter = (props) => {
	const [count, setCount] = useState(cache[props.name] ?? 0)
	const countRef = useRef(count)
	const increment = () => {
    setCount(count + 1)
    countRef.current++
  }

	useEffect(() => {
    setCount(cache[props.name] ?? 0)
		countRef.current = cache[props.name] ?? 0

    return () => {
      cache[props.name] = countRef.current
    };
  }, [props.name])

	...
```

Now the cleanup function for the `useEffect` will always use the most up-to-date data for `count` when setting the cache's value. This is the approach used within [the codesandbox link](https://codesandbox.io/s/cocky-water-7rz6g) from before for the "Replicating Re-Rendering" section.

## Wrapping Up

This post was born from the mistakes that my colleagues and I have made in the past. I hope this has helped you understand React a little better and I welcome you to share anything you've learned from prior mistakes!

Finally, if you've noticed any issues above, please let me know.