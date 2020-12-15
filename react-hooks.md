# Optimizing React Hooks Based Applications

## A brief overview of hooks
Announced by Sophie Alpert and Dan Abramov at React Conf 2018, React Hooks are a way to bring the capabilities of functional components closer to those of those of Class components. Additionally, the React team pointed out functional components are much easier for them to optimize, whereas class components are more difficult to optimize. Eventually, React Hooks were released in version 16.8

## Problems with hooks

As the software engineering field matures we are finding that code heavy in side effects tends to also be heavy in defects. While this isn't true for all code bases, but practice has taught us that we need to be cautious with our use of side effects in code. 

To combat this issue many have started to follow the pure function concept from functional programming. Pure functions are functions that encapsulate all of their state and has no side effects. However, at some point all applications will have to include some side effects whether its making api calls or saving data to a database. 

Unfortunately for those that strive to build their applications mostly with pure functions, React Hooks violate the rules of no side effects. That is to say that React hooks explicitly externalize state and inherently are side effects. 

So you may ask yourself if side effects are so bad, then why are Hooks a good thing? I asked myself that same question and grew a decent hatred towards them after seeing them misused. It is this misuse of side effects that can make them bad. 

## Simply using hooks wrong.
The simplistic api for hooks makes getting up and running trivial. However, there are rules about their usage that need to be followed.Unfortunately because hooks look like standard javascript, its easy to misuse hooks unless you have read the online documentation for their usage.

There are two main rules to react hooks that you need to follow to ensure proper usage. First, you must never call a hook from inside a regular javascript function, they must always be part of your component. Secondly, never call a hook inside a loop, condition or nested function. Failure to follow these two rules can have negative side effects because React requires that hooks always be used in the same order. 

This means this is bad
```javascript
const MyComponent = (props) => {
    const [name, setName] = useState("");
    if(props.foo === "bar") {
        const [baz, setBaz] = useState("baz"); 
    }
    ...
}
```

This is because we are defining a hook inside of a conditional, therefore there is no way that we can be certain that every time this component loads, that all of the hooks would be loaded in the same order. It would be better to define this component as follows: 

```javascript
const MyComponent = (props) => {
    const [name, setName] = useState("");
    const [baz, setBaz] = useState(props.foo === "bar" ? : "baz" : undefined);
}
```

Unfortunately both are valid code javascript, however the former breaks the rules that React has set out for their usage. We can enforce these rules by using the react hooks eslint plugin `eslint-plugin-react-hooks`

## Not returning a cleanup function from useEffect

The typical example of where the `useEffect` hook can be used is a class component that would load something during the `componentDidMount` lifecycle call. While it may not always be the case its also common to have clean up code in the `componentWillUnmount` lifecycle call. In order to replicate this functionality, you need to now return a function from useEffect that will be called when the component is us mounted. 

For example, let say you are subscribing to receiving server side events (SSE)'s 

In a class component you would do something like this
```typescript
class EventList { 
    public EventList(streamUrl: string) {
        this.streamUrl = streamUrl;
        this.appendEvent = this.appendEvent.bind(this);
        this.state = {events: []}
    }

    componentDidMount() {
        let stream = new EventSource(this.streamUrl);
        stream.onmessage = (event) => events.push(event.data);
    }

    appendEvent(event) {
        this.setState = { events: [...this.state.events, event]};
    }

    componentWillUnmount() {
        if(stream) {
            stream.close();
        }
    }
    render() {
        //do the render stuff
    }
}
```

In this example we would be connecting to service providing ServerSentEvents and consuming them by adding them to the `events` array. This is all fairly straight forward, so lets convert this to a functional component with hooks. 

```typescript
const EventList = (streamUrl: string) => {
    const [events, setEvents] = useState([]);
    useEffect(() => {
        const stream = new EventSource(streamUrl);
        stream.onMessage = (event) => {
            setEvent(events => [...events, event]);
        }
    })
    return (
        //do the render stuff
    )
}
```

Thats a heck of a lot shorter, but has a __major__ problem, every time this component re-renders, its going to create a new event source, and start listening. While a few may not be bad, there eventually will be a point where the zombie connection to the event source will prevent other consumers from connecting to the event source. To fix this we need to clean up our event source like we did in the class component. However, we have no `componentWillUnmount` lifecycle function since were in a functional component. So how do we do this? 

Well, its actually quite easy. The `useEffect` function as shown returns nothing, however the react `useEffect` allows you to return a function, that will inturn be used to clean up any resources used when the component is unmounted. 

To make this better we would only have to modify the component to be like this
```javascript
const EventList = (streamUrl: string) => {
    const [events, setEvents] = useState([]);
    useEffect(() => {
        const stream = new EventSource(streamUrl);
        stream.onMessage = (event) => {
            setEvent(events => [...events, event]);
        }
        return () => stream.close();
    })
    
    return (
        //do the render stuff
    )
}
```

## Too many renders. 

Regardless of whether or not we use React hooks in our application, re-rendering of components is one of the biggest killers of application performance. While the virtual DOM employed by React is a great performance improvement over legacy frameworks and libraries such as Angular 1 and jquery, understanding how re-renders impact performance is tantamount to creating performant React applications.

In a class based component, we can control re-rendering through the `shouldComponentUpdate` life cycle. Unfortunately there is no parallel to this in Functional Components. So how do we prevent unnecessary renders with hooks?


### The dependency array
The `useEffect` and `useCallback` hooks both have an optional argument to accept an array of dependencies that should be monitored to determine if the component needs to re-render. The confusion that can pop up here is that the dependency array is completely optional, and leaving it out means that the component should _always_ re-render! 

If we go back to our stream component from earlier, we can see that the `useEffect` hook that we utilize to connect to the server side event stream doesn't use a dependency array, therefore this will attempt to reconnect to the stream on every re render of this component. To prevent this we can pass an empty dependency array to the useEffect hook to signify that we never want this to be re-evaluated on re-render.

```javascript
const MyStreamComponent = (streamUrl) => {
    const [events, setEvents] = useState([]);
    useEffect(() => {
        const stream = new EventSource(streamUrl);
        stream.onMessage = (event) => {
            setEvent(events => [...events, event]);
        }
        return () => {
            stream.close();
        }
    }, [])
    return (
        //do the render stuff
    )
}
```

Now lets add some details to our return to have some details
```javascript
const Item = React.memo(({ event, markEventAcknowledged }) => {
  const { acknowledged, message } = event.data;
  const backgroundColor = acknowledged ? "green" : "red"

  return (
    <li style={{ backgroundColor, color: "white", margin: "2px" }}>
      <div>Event {event.lastEventId}</div>
      <div>Message {message}</div>
      <div>{
        !acknowledged &&
        <button onClick={() => markEventAcknowledged(event.lastEventId)}>
          Click to acknowledge.
        </button>
      }
      </div>
    </li>)
})

const EventList = () => {
  const [events, setEvents] = useState([]);
  useEffect(() => {
    const stream = new EventGenerator(streamUrl);
    stream.onmessage = event => setEvents(events => [...events, event]);
    return () => stream.close();
  }, [])

  const markEventAcknowledged = (id) => {
    setEvents(events => {
      const index = events.findIndex(event => event.lastEventId === id);
      if (index >= 0) {
        const event = { ...events[index] };
        event.data.acknowledged = true;
        return [
          ...events.slice(0, index),
          event,
          ...events.slice(index + 1)
        ];
      }
      return events;
    })
  };

  return (
    <div>
      <h1>Events</h1>
      <ul style={{ padding: "2px" }}>
        {
          events.map((event, idx) => (
            <Item key={`event-${idx}`} event={event} markEventAcknowledged={markEventAcknowledged} />)
          )
        }
      </ul>
    </div>
  );
}

```
If you noticed, we wrapped the definition of the `<Item>` component in a call to `React.memo`. `React.memo` is a performance hint that allows you to force a Pure Component to only be rendered when its props change. In a small component like `Item`, the performance gain can be negligible, however if it was doing a lot of complex work like formatting values for display etc, its worthwhile to consider. 

Unfortunately though, memoizing the `Item` component is not enough to prevent the `Item` components from re-rendering every time the `EventList` re-renders. This is because of the `markEventAcknowledged` function. Every time the `EventList` is recreated, a new copy of `markEventAcknowledged` is created. And, if you follow the rules for equality in javascript, a function can only the equal to itself. For example lets say we have a curried function `sumBuilder` defined as 
```javascript
const sumBuilder = (a) => (b) => a + b;
```

now if id create two copies of sumBuilder

```javascript
const adding1 = sumBuilder(1);
const add1 = sumBuilder(1);
```
now if you were to compare both of those, they would not be equal, even though they do the same work. So, every time we pass our props to the `Item` component, the `markEventAcknowledged` function is different and will force the component to be re-rendered.

So how do we fix this? This is where another hook, `useCallback` comes in handy. This is a simple call back that allows us to memoize the construction of the `markEventAcknowledged` function. To use the `useCallback` hook, we need to wrap the construction of the call back in the hook like this

```javascript 
  const markEventAcknowledged = useCallback((id) => {
    setEvents(events => {
      const index = events.findIndex(event => event.lastEventId === id);
      if (index >= 0) {
        const event = { ...events[index] };
        event.data.acknowledged = true;
        return [
          ...events.slice(0, index),
          event,
          ...events.slice(index + 1)
        ];
      }
      return events;
    })
  }, []);
```

Now, every time out `EventList` re-renders, which will happen every time a new event occurs, our `Item` components will receive the memoized version of the `markEventAcknowledged` function and which will be unchanging and allow our `Item` components to only rerender if their event changes. 

## Conclusion

So now that we have a better understanding of hooks and how to optimize their usage, its important to recall how to get the most out of them. First, use the eslint plugins to help prevent common hooks pitfalls. Secondly, always include a dependency array, allowing your hook to always update on re-render is rarely what you want. Use all of your hooks, including `useCallback` to your advantage. And finally, don't go overboard trying to performance tune your application. If your components are lightweight enough, the cost of letting them just re-render isn't going to be detrimental. Always, optimize when the need to do so comes up, not because you think it will be a problem. 
