---
title: create ⚛️
description: How to create stores
nav: 26
---

`create` lets you create a React Hook with API utilities attached.

```js
create(stateCreatorFn)
```

- [Reference](#reference)
  - [Signature](#create-signature)
- [Usage](#usage)
  - [Updating state based on previous state](#updating-state-based-on-previous-state)
  - [Updating Primitives in State](#updating-primitives-in-state)
  - [Updating Objects in State](#updating-objects-in-state)
  - [Updating Arrays in State](#updating-arrays-in-state)
  - [Updating state with no store actions](#updating-state-with-no-store-actions)
  - [Subscribing to state updates](#subscribing-to-state-updates)
- [Troubleshooting](#troubleshooting)
  - [I’ve updated the state, but the screen doesn’t update](#ive-updated-the-state-but-the-screen-doesnt-update)

## Reference

### `create` Signature

```ts
create<T>()(stateCreatorFn: StateCreator<T, [], []>): UseBoundStore<StoreApi<T>>
```

#### Parameters

- `stateCreatorFn`: A function that takes `set` function, `get` function and `api` as arguments.
  Usually, you will return an object with the methods you want to expose.

#### Returns

`create` returns a React Hook with API utilities, `setState`, `getState`, `getInitialState` and
`subscribe`, attached. It lets you return data that is based on current state, using a selector
function. It should take a selector function as its only argument.

## Usage

### Updating state based on previous state

To update a state based on previous state we should use **updater functions**. Read more
about that [here](https://react.dev/learn/queueing-a-series-of-state-updates).

This example shows how you can support **updater functions** for your **actions**.

```tsx
import { create } from 'zustand'

type AgeStoreState = { age: number }

type AgeStoreActions = {
  setAge: (
    nextAge:
      | AgeStoreState['age']
      | ((currentAge: AgeStoreState['age']) => AgeStoreState['age']),
  ) => void
}

type AgeStore = AgeStoreState & AgeStoreActions

const useAgeStore = create<AgeStore>()((set) => ({
  age: 42,
  setAge: (nextAge) => {
    set((state) => ({
      age: typeof nextAge === 'function' ? nextAge(state.age) : nextAge,
    }))
  },
}))

export default function App() {
  const age = useAgeStore((state) => state.age)
  const setAge = useAgeStore((state) => state.setAge)

  function increment() {
    setAge((currentAge) => currentAge + 1)
  }

  return (
    <>
      <h1>Your age: {age}</h1>
      <button
        onClick={() => {
          increment()
          increment()
          increment()
        }}
      >
        +3
      </button>
      <button
        onClick={() => {
          increment()
        }}
      >
        +1
      </button>
    </>
  )
}
```

### Updating Primitives in State

State can hold any kind of JavaScript value. When you want to update built-in primitive values like
numbers, strings, booleans, etc. you should directly assign new values to ensure updates are applied
correctly, and avoid unexpected behaviors.

> [!NOTE]
> By default, `set` function performs a shallow merge. If you need to completely replace the state
> with a new one, use the `replace` parameter set to `true`

```tsx
import { create } from 'zustand'

type XStore = number

const useXStore = create<XStore>()(() => 0)

export default function MovingDot() {
  const x = useXStore()
  const setX = (nextX: number) => {
    useXStore.setState(nextX, true)
  }
  const position = { y: 0, x }

  return (
    <div
      onPointerMove={(e) => {
        setX(e.clientX)
      }}
      style={{
        position: 'relative',
        width: '100vw',
        height: '100vh',
      }}
    >
      <div
        style={{
          position: 'absolute',
          backgroundColor: 'red',
          borderRadius: '50%',
          transform: `translate(${position.x}px, ${position.y}px)`,
          left: -10,
          top: -10,
          width: 20,
          height: 20,
        }}
      />
    </div>
  )
}
```

### Updating Objects in State

Objects are **mutable** in JavaScript, but you should treat them as **immutable** when you store
them in state. Instead, when you want to update an object, you need to create a new one (or make a
copy of an existing one), and then set the state to use the new object.

By default, `set` function performs a shallow merge. For most updates where you only need to modify
specific properties, the default shallow merge is preferred as it's more efficient. To completely
replace the state with a new one, use the `replace` parameter set to `true` with caution, as it
discards any existing nested data within the state.

```tsx
import { create } from 'zustand'

type PositionStoreState = { x: number; y: number }

type PositionStoreActions = {
  setPosition: (nextPosition: Partial<PositionStoreState>) => void
}

type PositionStore = PositionStoreState & PositionStoreActions

const usePositionStore = create<PositionStore>()((set) => ({
  x: 0,
  y: 0,
  setPosition: (nextPosition) => {
    set(nextPosition)
  },
}))

export default function MovingDot() {
  const [position, setPosition] = usePositionStore((state) => [
    { x: state.x, y: state.y },
    state.setPosition,
  ])

  return (
    <div
      onPointerMove={(e) => {
        setPosition({
          x: e.clientX,
          y: e.clientY,
        })
      }}
      style={{
        position: 'relative',
        width: '100vw',
        height: '100vh',
      }}
    >
      <div
        style={{
          position: 'absolute',
          backgroundColor: 'red',
          borderRadius: '50%',
          transform: `translate(${position.x}px, ${position.y}px)`,
          left: -10,
          top: -10,
          width: 20,
          height: 20,
        }}
      />
    </div>
  )
}
```

### Updating Arrays in State

Arrays are mutable in JavaScript, but you should treat them as immutable when you store them in
state. Just like with objects, when you want to update an array stored in state, you need to create
a new one (or make a copy of an existing one), and then set state to use the new array.

By default, `set` function performs a shallow merge. To update array values we should assign new
values to ensure updates are applied correctly, and avoid unexpected behaviors. To completely
replace the state with a new one, use the `replace` parameter set to `true`.

> [!IMPORTANT]
> We should prefer immutable operations like: `[...array]`, `concat(...)`, `filter(...)`,
> `slice(...)`, `map(...)`, `toSpliced(...)`, `toSorted(...)`, and `toReversed(...)`, and avoid
> mutable operations like `array[arrayIndex] = ...`, `push(...)`, `unshift(...)`, `pop(...)`,
> `shift(...)`, `splice(...)`, `reverse(...)`, and `sort(...)`.

```tsx
import { create } from 'zustand'

type PositionStore = [number, number]

const usePositionStore = create<PositionStore>()(() => [0, 0])

export default function MovingDot() {
  const [x, y] = usePositionStore()
  const setPosition: typeof usePositionStore.setState = (nextPosition) => {
    usePositionStore.setState(nextPosition, true)
  }
  const position = { x, y }

  return (
    <div
      onPointerMove={(e) => {
        setPosition([e.clientX, e.clientY])
      }}
      style={{
        position: 'relative',
        width: '100vw',
        height: '100vh',
      }}
    >
      <div
        style={{
          position: 'absolute',
          backgroundColor: 'red',
          borderRadius: '50%',
          transform: `translate(${position.x}px, ${position.y}px)`,
          left: -10,
          top: -10,
          width: 20,
          height: 20,
        }}
      />
    </div>
  )
}
```

### Updating state with no store actions

Defining actions at module level, external to the store have a few advantages like: it doesn't
require a hook to call an action, and it facilitates code splitting.

> [!NOTE]
> The recommended way is to colocate actions and states within the store (let your actions be
> located together with your state).

```tsx
import { create } from 'zustand'

const usePositionStore = create<{
  x: number
  y: number
}>()(() => ({ x: 0, y: 0 }))

const setPosition: typeof usePositionStore.setState = (nextPosition) => {
  usePositionStore.setState(nextPosition)
}

export default function MovingDot() {
  const position = usePositionStore()

  return (
    <div
      style={{
        position: 'relative',
        width: '100vw',
        height: '100vh',
      }}
    >
      <div
        style={{
          position: 'absolute',
          backgroundColor: 'red',
          borderRadius: '50%',
          transform: `translate(${position.x}px, ${position.y}px)`,
          left: -10,
          top: -10,
          width: 20,
          height: 20,
        }}
        onMouseEnter={(event) => {
          const parent = event.currentTarget.parentElement
          const parentWidth = parent.clientWidth
          const parentHeight = parent.clientHeight

          setPosition({
            x: Math.ceil(Math.random() * parentWidth),
            y: Math.ceil(Math.random() * parentHeight),
          })
        }}
      />
    </div>
  )
}
```

### Subscribing to state updates

By subscribing to state updates, you register a callback that fires whenever the store's state
updates. We can use `subscribe` for external state management.

```tsx
import { useEffect } from 'react'
import { create } from 'zustand'

type PositionStoreState = { x: number; y: number }

type PositionStoreActions = {
  setPosition: (nextPosition: Partial<PositionStoreState>) => void
}

type PositionStore = PositionStoreState & PositionStoreActions

const usePositionStore = create<PositionStore>()((set) => ({
  x: 0,
  y: 0,
  setPosition: (nextPosition) => {
    set(nextPosition)
  },
}))

export default function MovingDot() {
  const [position, setPosition] = usePositionStore((state) => [
    { x: state.x, y: state.y },
    state.setPosition,
  ])

  useEffect(() => {
    const unsubscribePositionStore = usePositionStore.subscribe(({ x, y }) => {
      console.log('new position', { position: { x, y } })
    })

    return () => {
      unsubscribePositionStore()
    }
  }, [])

  return (
    <div
      style={{
        position: 'relative',
        width: '100vw',
        height: '100vh',
      }}
    >
      <div
        style={{
          position: 'absolute',
          backgroundColor: 'red',
          borderRadius: '50%',
          transform: `translate(${position.x}px, ${position.y}px)`,
          left: -10,
          top: -10,
          width: 20,
          height: 20,
        }}
        onMouseEnter={(event) => {
          const parent = event.currentTarget.parentElement
          const parentWidth = parent.clientWidth
          const parentHeight = parent.clientHeight

          setPosition({
            x: Math.ceil(Math.random() * parentWidth),
            y: Math.ceil(Math.random() * parentHeight),
          })
        }}
      />
    </div>
  )
}
```

## Troubleshooting

### I’ve updated the state, but the screen doesn’t update

In the previous example, the `position` object is always created fresh from the current cursor
position. But often, you will want to include existing data as a part of the new object you’re
creating. For example, you may want to update only one field in a form, but keep the previous
values for all other fields.

These input fields don’t work because the `onChange` handlers mutate the state:

```tsx
import { create } from 'zustand'

type PersonStoreState = {
  firstName: string
  lastName: string
  email: string
}

type PersonStoreActions = {
  setPerson: (nextPerson: Partial<PersonStoreState>) => void
}

type PersonStore = PersonStoreState & PersonStoreActions

const usePersonStore = create<PersonStore>()((set) => ({
  firstName: 'Barbara',
  lastName: 'Hepworth',
  email: 'bhepworth@sculpture.com',
  setPerson: (nextPerson) => {
    set(nextPerson)
  },
}))

export default function Form() {
  const [person] = usePersonStore((state) => [
    {
      firstName: state.firstName,
      lastName: state.lastName,
      email: state.email,
    },
    state.setPerson,
  ])

  function handleFirstNameChange(e) {
    person.firstName = e.target.value
  }

  function handleLastNameChange(e) {
    person.lastName = e.target.value
  }

  function handleEmailChange(e) {
    person.email = e.target.value
  }

  return (
    <>
      <label style={{ display: 'block' }}>
        First name:
        <input value={person.firstName} onChange={handleFirstNameChange} />
      </label>
      <label style={{ display: 'block' }}>
        Last name:
        <input value={person.lastName} onChange={handleLastNameChange} />
      </label>
      <label style={{ display: 'block' }}>
        Email:
        <input value={person.email} onChange={handleEmailChange} />
      </label>
      <p>
        {person.firstName} {person.lastName} ({person.email})
      </p>
    </>
  )
}
```

For example, this line mutates the state from a past render:

```tsx
person.firstName = e.target.value
```

The reliable way to get the behavior you’re looking for is to create a new object and pass it to
`setPerson`. But here you want to also copy the existing data into it because only one of the
fields has changed:

```ts
setPerson({
  firstName: e.target.value, // New first name from the input
})
```

> [!NOTE]
> We don’t need to copy every property separately due to `set` function performing shallow merge by
> default.

Now the form works!

Notice how you didn’t declare a separate state variable for each input field. For large forms,
keeping all data grouped in an object is very convenient—as long as you update it correctly!

```tsx {35,39,43}
import { create } from 'zustand'

type PersonStoreState = {
  firstName: string
  lastName: string
  email: string
}

type PersonStoreActions = {
  setPerson: (nextPerson: Partial<PersonStoreState>) => void
}

type PersonStore = PersonStoreState & PersonStoreActions

const usePersonStore = create<PersonStore>()((set) => ({
  firstName: 'Barbara',
  lastName: 'Hepworth',
  email: 'bhepworth@sculpture.com',
  setPerson: (nextPerson) => {
    set(nextPerson)
  },
}))

export default function Form() {
  const [person, setPerson] = usePersonStore((state) => [
    {
      firstName: state.firstName,
      lastName: state.lastName,
      email: state.email,
    },
    state.setPerson,
  ])

  function handleFirstNameChange(e) {
    setPerson({ firstName: e.target.value })
  }

  function handleLastNameChange(e) {
    setPerson({ lastName: e.target.value })
  }

  function handleEmailChange(e) {
    setPerson({ email: e.target.value })
  }

  return (
    <>
      <label style={{ display: 'block' }}>
        First name:
        <input value={person.firstName} onChange={handleFirstNameChange} />
      </label>
      <label style={{ display: 'block' }}>
        Last name:
        <input value={person.lastName} onChange={handleLastNameChange} />
      </label>
      <label style={{ display: 'block' }}>
        Email:
        <input value={person.email} onChange={handleEmailChange} />
      </label>
      <p>
        {person.firstName} {person.lastName} ({person.email})
      </p>
    </>
  )
}
```