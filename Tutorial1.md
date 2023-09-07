
# React Hooks + THREE.js

## Motivation
With THREE.js, you need to instantiate several objects each time you use it. You need to create a Scene, a Camera, a Renderer, some 3D objects, and maybe a canvas. Beyond that, if you need controls, maybe you setup [`DAT.gui`](https://github.com/dataarts/dat.gui) like the examples use.

The Renderer needs a reference to the Camera and the Scene. You need a reference to the Scene to add any 3D objects you create. It's up to you how to structure everything cleanly.

In contrast, React devs are now used to the ease and simplicity of using `create-react-app` to try things out. It would be great to have a React component you could throw some random THREE.js code into and have it work!

Thinking further, imagine if you could somehow treat THREE.js scene objects like React components! What if you were able to write THREE.js code like this:

```html
<Scene>
  <CameraControls />
  <Cube h={12} w={12} d={12} />
</Scene>
```


The pattern I explain below aims to allow this in a very minimally obtrusive way.

## React Hooks

A few features in React 16.x make using THREE.js (or any external library) in a React app a lot cleaner. Those are [`forwardRef`](https://reactjs.org/docs/forwarding-refs.html), and some of the new, experimental [Hooks](https://reactjs.org/docs/hooks-intro.html): [`useRef`](https://reactjs.org/docs/hooks-reference.html#useref), and most importantly [`useEffect`](https://reactjs.org/docs/hooks-reference.html#useeffect).

Read the docs, but basically what the features allow is the ability to create function components which:

- Use `useEffect` to configurably run arbitrary JS (such as calling a third party library)
- Use `useRef` to store a reference to any arbitrary value (such as third-party classes)
- Combine the above with `forwardRef` to create a component that makes any arbitrary reference available to it's parent

Here is how you could use these hooks to make a `Cube` functional component:

```js
function Cube = (props) {
  const entityRef = useRef()
  const scene = getScene()

  useEffect(
    () => {
      entityRef.current = createThreeJSCube(scene, props)
      return () => {
        cleanupThreeJSCube(scene)
      }
    },
    [],
  )

  return null
}
```

Ignoring for now where we're getting `scene` from, what we've done is created a React component, which, when mounted, calls `createThreeJSCube` and stores a reference to the return value, and when unmounted, calls `cleanupThreeJSCube`. It renders `null`, so doesn't effect the DOM; it *only* has side effects. Interesting.

In case you haven't read up on `useEffect` yet, the 2nd argument is the hook's dependencies, by specifying an empty array, we're indicating this hook doesn't have dependencies and should only be run once. Omitting the argument indicates it should be run on every render, and adding references into the array will cause the hook to run only when the references have changed.

Using this knowledge, we can add a second hook to our `Cube` component to run some effects when props change. Since we stored the output from our THREE.js code into `entityRef.current`, we can now access it from this other hook:

```js
function Cube(props) {
  ...
  useEffect(
    () => {
      updateCubeWithProps(entityRef.current, props)
    },
    [props]
  )
}
```

We now have a React component which adds a 3D object to a THREE.js scene, alters the object when it gets new props, and cleans itself up when we unmount it. Awesome! Now we just need to make `scene` available in our component so that it actually works.

## `forwardRef`

Before discussing how we get `scene` available in our component, let's discuss another newer React feature that will help us setup THREE.js in the way we need: `forwardRef`. Remember, before we can even get to adding 3D objects to the `scene`, we still need to setup our canvas, renderer, camera, and all of that.

Consider the fact that, in THREE.js, several things need reference to the `canvas` element. In more vanilla usages this `canvas` is created by THREE code itself, but we want more control, so we're going to render it from a React component so we can encapsulate resize actions and anything else specific to the canvas in that component. Now we have a problem though, in that, the DOM element is only available in that component. How do we solve this? `forwardRef`! With `forwardRef`, we can create a `Canvas` component, that renders a canvas element, and forwards it's ref to it. So anyone for anyone rendering `<Canvas ref={myRef} />`, `myRef` will point to the `canvas` HTML element itself, not the `Canvas` React component. Cool!

```js
const Canvas = forwardRef((props, ref) => {
  ...
  return (
    <canvas ref={ref} ... />
  )
})
```

Also remember that, from the React docs, `ref`s are not just for DOM element references! We can set and forward a `ref` to any value. This means we can use `forwardRef` along with `useEffect` to create React components for our THREE.js Renderer and Camera, that make their THREE.js objects available to their parents. Here's how it looks:

```js
const Camera = forwardRef((props, ref) => {
  ...
  useEffect(
    () => {
      const camera = new THREE.PerspectiveCamera(...)
      ...
      ref = camera
    },
    [],
  )
})
```

Now, for a component rendering `<Camera ref={myRef} />`, `myRef` will be the actual THREE.js camera instance! Additionally, we can add an effect that triggers on the `aspectRatio` prop, so that the camera updates itself when props change (such as on window resize):

```js
const Camera = forwardRef((props, ref) => {
  ...
  useEffect(
    () => {
      ref.current.aspect = aspectRatio
      ref.current.updateProjectionMatrix()
    },
    [aspectRatio],
  )
  ...
})
```

We'll use the same technique to create a `Renderer` component. Then, we'll be able to create a `SceneManager` component that has references to all the THREE.js objects we need to render a scene! Awesome!

## `SceneManager` Component

Using the above techniques, we can create a `SceneManager` component that has `ref`s to everything we need to use THREE.js: `Camera`, `Renderer`, `scene`, and a `canvas`. Remember though, we still need to make `scene` available to child components. For this, we'll have `SceneManager` render a `Context.Provider` with the value. Then, we can use the `useContext` hook to access `scene` in our React/THREE.js components:

```js
function Cube = (props) {
  const context = useContext(SceneContext)
  const { scene } = context
  ...
}
```

We'll add `canvas`, and `camera` to the `Provider` value also, since it will be useful to access those objects sometimes, such as for setting up camera controls.

In a nutshell, `SceneManager` abstracts away the base-level THREE.js tasks you need to do to before you can add 3D objects. Here's how the return value might look:

```html
<SceneContext.Provider value={{
  scene,
  camera,
  canvas,
}}>
  <Canvas ref={canvasRef} />
  <Camera ref={cameraRef} {...cameraProps} />
  <Renderer ref={rendererRef} {...rendererProps} >
  { props.children }
</SceneContext.Provider>
```


## `useScene` Custom Hook
Let's look at creating a custom hook to setup our React/THREE.js components. Basically, we want to abstract away what we've done with `Cube` to be able to use it generically. Here are the high level steps of what we did:

In a functional component:

- Get the `scene` from somewhere
- Use `useRef` to initialize a placeholder called `entityRef` that stores the 3D object
- Use `useEffect` to run code on mount that instantializes the 3D object, assigns it to `entityRef`, adds it to the `scene`, and returns a cleanup function that removes it from the scene
- Use `useEffect` to run code when `props` change that accesses `entityRef` and updates the 3D object, reflecting the changes in React onto the 3D

We'll only abstract out the first three bullets above, considering that handling `props` changing will be specific to each component, and possibly not needed in all cases. We'll also account for the fact that you may want to use some of the props inside the setup hook. Here's how the method signature of our custom hook might look:
```
useScene(props, setup, [destroy])
```
Arguments:
- `props` _(React props)_: Reference to the component's props. Needed to pass the props to `setup` function.
- `setup` _(Function)_: This function will be called with 2 arguments, `scene` and `props`. This is where you setup the 3D object. What you return here will be internally stored and available later through the getter returned by `useScene`.
- `destroy` _(Function)_: This function will be called when the component is unmounted. It gets called with 2 arguments: `scene` (same as `setup` was called with) and a reference to whatever was returned by `setup`. If not passed, `scene.remove` is called with the return value of `setup` (default).

Returns: _(Function)_

`useScene` returns a function which, when called, returns a reference to the return value of the function passed to it as the `setup` argument. Whatever is returned from `setup` is stored internally inside the hook for you to access with this function whenever you need to, ie when props change.

Here's the code for our custom hook:

```js
const useScene = (props, setup, destroy) => {
  const entityRef = useRef()
  const getEntity = () => entityRef.current

  const context = useContext(SceneContext)
  const { scene } = context

  useEffect(
    () => {
      entityRef.current = setup(scene, props)

      return () => {
        if (destroy) {
          return destroy(scene, getEntity())
        }
        scene.remove(getEntity())
      }
    },
    [],
  )

  return getEntity
}
```

Here's how you'd use it to add a simple grid object to the scene:

```js
const Grid = props => {
  useScene(props, (scene) => {
    const grid = new THREE.GridHelper(1000, 100);
    scene.add(grid);

    return grid;
  });

  return null;
};
```

Notice a few things here:
- We didn't pass `destroy` param, so `useScene` will just call `scene.remove` with whatever we return from `setup` when the component is unmounted
- The component returns `null`. This is required for React not to complain "hey, your component didn't render anything!".
- We don't care about props changing, so we don't store the return value of `useScene` (which gives us access to the `setup` return value).

If we did care about props changing, we could store the reference returned from `useScene` and use it in another effect that triggers off props changing:

```js
const Grid = props => {
  const { color } = props
  const getEntity = useScene(...)

  useEffect(
    () => {
      getEntity().material.color.set(props.color);
    },
    [color],
  )
  ...
};
```

If we wanted to do something specific on unmount, we can pass a `destroy` function as the 3rd argument. Perhaps our component is complex and has several THREE.js objects and our `setup` function returned an object containing all of them:

```js
const ComplexThreeComponent = props => {
  const useScene(
    props,
    (scene, props) => {
      ...
      return {
        arms,
        body,
        leg,
      }
    },
    (scene, entity) => {
      const { arms, body, leg } = entity
      scene.remove(arms)
      scene.remove(body)
      scene.remove(leg)
    }
  });
  ...
};
```
