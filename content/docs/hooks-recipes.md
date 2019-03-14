---
id: hooks-recipes
title: Hooks Recipes
permalink: docs/hooks-recipes.html
prev: hooks-faq.html
---

*Hooks* are a new addition in React 16.8. They let you use state and other React features without writing a class.

This is a collection of recipes that may help you start "thinking in Hooks". Feel free to play with them or copy and paste them into your project or tweak them to your needs.

## Data Fetching

## Subscriptions

Sometimes React components need to subscribe to an external data source. Handling all the possible edges in a performant way can be challenging, which is why people often use dedicated third-party libraries for connecting to data. The `useSubscription` Hook below simplifies that. It lets you subscribe to an external data source while handling all the edge cases. It is also compatible with the upcoming concurrent rendering mode.

```js
// Hook used for safely managing subscriptions in concurrent mode.
//
// In order to avoid removing and re-adding subscriptions each time this hook is called,
// the parameters passed to this hook should be memoized in some way–
// either by wrapping the entire params object with useMemo()
// or by wrapping the individual callbacks with useCallback().
export function useSubscription<Value>({
  // (Synchronously) returns the current value of our subscription.
  getCurrentValue,

  // This function is passed an event handler to attach to the subscription.
  // It should return an unsubscribe function that removes the handler.
  subscribe,
}: {|
  getCurrentValue: () => Value,
  subscribe: (callback: Function) => () => void,
|}): Value {
  // Read the current value from our subscription.
  // When this value changes, we'll schedule an update with React.
  // It's important to also store the hook params so that we can check for staleness.
  // (See the comment in checkForUpdates() below for more info.)
  const [state, setState] = useState({
    getCurrentValue,
    subscribe,
    value: getCurrentValue(),
  });

  // If parameters have changed since our last render, schedule an update with its current value.
  if (
    state.getCurrentValue !== getCurrentValue ||
    state.subscribe !== subscribe
  ) {
    setState({
      getCurrentValue,
      subscribe,
      value: getCurrentValue(),
    });
  }

  // It is important not to subscribe while rendering because this can lead to memory leaks.
  // (Learn more at reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects)
  // Instead, we wait until the commit phase to attach our handler.
  //
  // We intentionally use a passive effect (useEffect) rather than a layout effect.
  // so that we don't stretch the commit phase.
  // This also has an added benefit when multiple components are subscribed to the same source:
  // It allows each of the event handlers to safely schedule work
  // without potentially interfering with another subscription.
  // (Learn more at https://codesandbox.io/s/k0yvr5970o)
  useEffect(
    () => {
      let didUnsubscribe = false;

      const checkForUpdates = () => {
        // It's possible that this callback will be invoked even after being unsubscribed,
        // if it's removed as a result of a subscription event/update.
        // In this case, React will log a DEV warning about an update from an unmounted component.
        // We can avoid triggering that warning with this check.
        if (didUnsubscribe) {
          return;
        }

        setState(prevState => {
          // Ignore values from stale sources!
          // Since we subscribe an unsubscribe in a passive effect,
          // it's possible that this callback will be invoked for a stale (previous) subscription.
          // This check avoids scheduling an update for that stale subscription.
          if (
            prevState.getCurrentValue !== getCurrentValue ||
            prevState.subscribe !== subscribe
          ) {
            return prevState;
          }

          // Some subscriptions will auto-invoke the handler, even if the value hasn't changed.
          // If the value hasn't changed, no update is needed.
          // Return state as-is so React can bail out and avoid an unnecessary render.
          const value = getCurrentValue();
          if (prevState.value === value) {
            return prevState;
          }

          return {...prevState, value};
        });
      };
      const unsubscribe = subscribe(checkForUpdates);

      // Because we're subscribing in a passive effect,
      // it's possible that an update has occurred between render and our effect handler.
      // Check for this and schedule an update if work has occurred.
      checkForUpdates();

      return () => {
        didUnsubscribe = true;
        unsubscribe();
      };
    },
    [getCurrentValue, subscribe],
  );

  // Return the current value for our caller to use while rendering.
  return state.value;
}
```

Here is an example of using the hook above to subscribe to a "notification service" containing information about the number of unread notifications:

```js{2-4,18}
function NotificationBadge({ notificationService }) {
  // In order to avoid removing and re-adding subscriptions each time this hook is called,
  // the parameters passed to this hook should be memoized with useMemo() or useCallback().
  const subscription = useMemo(
    () => ({
      getCurrentValue: () => notificationService.count,
      subscribe: callback => {
        notificationService.addEventListener("change", callback);
        return () =>
          notificationService.removeEventListener("change", callback);
      }
    }),

    // Re-subscribe if we ever get a new "notificationService" prop.
    [notificationService]
  );

  const count = useSubscription(subscription);

  return <label className="NotificationBadge">{count}</label>;
}
```