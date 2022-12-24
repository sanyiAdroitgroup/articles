# How can we use a global event bus in Vue 2 vs Vue 3?

### We can’t use it anymore in Vue 3. I mean, not in the form we are used to. This was one of the biggest breaking change in the new version of Vue. In this article we will discuss why we can’t use global event bus in a “traditional” way anymore and what are the alternatives.

---

## What is an event bus and how do we use it in Vue 2?

### Event bus in general

An event bus is a design pattern that is based on the publish/subscribe or pubsub pattern (we’ll talk about this pattern later on in this article). Event buses come in handy when we would like to communicate between decoupled components.

In a nutshell, the idea behind this pattern is that we can fire an event which we can subscribe to from another component. For example, we have an info button that should show a modal on click. But we register all of our modals in our root component. With an event bus, we can do this if we trigger a global event with the necessary payload and subscribe it from the root component.

However this solution seems easy and straightforward, it’s not recommended. Because in the long term it can cause serious troubles. It’s really hard to even detect which component fired the event. Since there aren't any restrictions to emitting the same event.

From the [official documentation](https://v3-migration.vuejs.org/breaking-changes/events-api.html#event-bus):

> In most circumstances, using a global event bus for communicating between components is discouraged. While it is often the simplest solution in the short term, it almost invariably proves to be a maintenance headache in the long term.
> 

### Event buses in Vue 2

Okay, time to see some code! Let’s implement a global event bus in Vue 2!

![https://media.giphy.com/media/sV2yEUHOlAWsssJfv8/giphy.gif](https://media.giphy.com/media/sV2yEUHOlAWsssJfv8/giphy.gif)

Before we get started, just as a reminder, every vm has these three (actually four, but we’re not going to use all) methods by default:

```jsx
// to emit the event
vm.$emit('event-name', payload)

// to listen for the event
vm.$on('event-name', callback)

// to remove listener for the given event
vm.$off('event-name', callback)
```

You can read more about these events [here](https://v2.vuejs.org/v2/api/#Instance-Methods-Events).

Let’s create another instance and define it on the prototype. This way we’ll able to use it from every component.

```jsx
import Vue from 'vue'
import App from './App.vue'

Vue.prototype.$eventBus = new Vue()

new Vue({
	render: h => h(App)
}).$mount('#app')
```

Now we are able to communicate between components. Move forward and create two components.

```jsx
<template>
  <div>
    <label>Name:</label>
    <input v-model="name" />
    <button @click="handleSubmitBtnClick">Submit</button>
  </div>
</template>

<script>
export default {
  name: "BaseInput",
  data: () => ({
    name: null,
  }),
  methods: {
    handleSubmitBtnClick() {
      this.$eventBus.$emit("form-submitted", this.name);
    },
  },
};
</script>
```

As you can see it’s just a dead simple form which contains an input with a submit button. We also added a click event listener to this button which will trigger a global event with the entered value.

Create another component which will listen for this event. 

```jsx
<template>
  <h1>Received name is: {{ receivedName }}</h1>
</template>

<script>
export default {
  name: "BaseInput",
  data: () => ({
    receivedName: null,
  }),
  methods: {
    handleFormSubmit(payload) {
      this.receivedName = payload;
    },
  },
  created() {
    this.$eventBus.$on("form-submitted", this.handleFormSubmit);
  },
  beforeDestroy() {
    this.$eventBus.$off("form-submitted", this.handleFormSubmit);
  },
};
</script>
```

The interesting part for us is in the `created` and `beforeDestory` lifecycle methods.

We subscribed to this event in the `created` lifecycle with the help of the `$on` method. So basically every time when the form is submitted the passed callback function will run. We also have to unsubscribe from this event before we destroy our component. Otherwise we’ll face with unexpected behaviours.  

## What about Vue 3?

Well the bad news is that if we want to migrate from version 2 to 3 and if we used the same or similar approach that we mentioned above then we’ll face with a big refactoring task. Because the case is that in Vue 3 `$on` , `$off` and `$once` have been removed. So we can’t use it in a way that we did before.

If you would like to read more about the reason behind this change you can get a clearer picture from this [conversation](https://news.ycombinator.com/item?id=23879595) or from this [blog post](https://tkacz.pro/vue-js-why-event-bus-is-bad-idea/).

Fortunately we have several solutions to solve this problem. 

### Mitt

Mitt is the [officially recommended](https://v3-migration.vuejs.org/breaking-changes/events-api.html#event-bus) library for handling events between components. 

The biggest advantage of this event emitter package is that is only 200 byte. 

Now let’s se how we can use it in a Vue 3 project!

First we have to install this package:

```
npm install --save mitt
```

Then in the main.js we provide it globally.

```jsx
import { createApp } from 'vue'
import App from './App.vue'
import mitt from "mitt"

const emitter = mitt()
const app = createApp(App)

app.provide('emitter', emitter)
app.mount('#app')
```

After this we are ready to use it in our components. Of course we have to also inject this emitter. 

```jsx
<template>
    <label>Name:</label>
    <input v-model="name" />
    <button @click="handleSubmitBtnClick">Submit</button>
</template>

<script setup>
import { ref, inject } from 'vue'
const emitter = inject('emitter')

const name = ref(null)

const handleSubmitBtnClick = () => {
    emitter.emit('form-submitted', name.value)
}
</script>
```

As you can see the basic idea is the same as we used in the Vue 2’s component. But unlike that we can’t access this event bus from the global instance. That’s why we have to inject it every time when we would like to use it. Only thing left is to subscribe to this event from `ComponentB.vue`.

```jsx
<template>
    <h1>Received name is: {{ receivedName }}</h1>
</template>

<script setup>
import { ref, inject, onBeforeUnmount } from 'vue'

const emitter = inject('emitter')
const receivedName = ref(null)

const handleFormSubmit = (payload) => {
    receivedName.value = payload
}

emitter.on('form-submitted', handleFormSubmit)

onBeforeUnmount(() => {
    emitter.off('form-submitted', handleFormSubmit)
})
</script>
```

For the first sight it could be weird that we didn’t do this subscription from a lifecycle. The reason of this is that in Vue 3 the `setup` method replaced the `created` lifecycle hook.

### Pub-sub pattern

The publish-subscribe (pub-sub) pattern is a software design pattern that allows for the separation of concerns between the publisher and the subscriber. In this pattern, the publisher is responsible for generating and disseminating information, while the subscriber is responsible for receiving and reacting to that information. In JavaScript, the pub-sub pattern is often used to implement event-driven programming, where an event emitter triggers an event and any number of event listeners can respond to it. 

Let’s use this pattern! Create a simple `Event` in class which will handle our events:

```jsx
class Event{
    constructor(){
        this.events = {};
    }

    on(eventName, fn) {
        this.events[eventName] = this.events[eventName] || [];
        this.events[eventName].push(fn);
    }

    off(eventName, fn) {
        if (this.events[eventName]) {
            for (var i = 0; i < this.events[eventName].length; i++) {
                if (this.events[eventName][i] === fn) {
                    this.events[eventName].splice(i, 1);
                    break;
                }
            };
        }
    }

    trigger(eventName, data) {
        if (this.events[eventName]) {
            this.events[eventName].forEach(function(fn) {
                fn(data);
            });
        }
    }
}

export default new Event();
```

From here on the methodology will be the same. We’ll import it in our component and use it. But to make it a little more exciting we’ll use the `<script>` syntax a different way as we did before:

```jsx
<template>
  <label>Name:</label>
  <input v-model="name" />
  <button v-on:click="handleSubmitBtnClick">Submit</button>
</template>

<script>
import { ref } from "vue";
import EventBus from "./composables/event";

export default {
  setup() {
    const name = ref(null);
    const handleSubmitBtnClick = () => {
      EventBus.trigger("form-submitted", name.value);
    };

    return {
      name,
      handleSubmitBtnClick,
    };
  },
};
</script>
```

```jsx
<template>
  <h1>Received name is: {{ receivedName }}</h1>
</template>

<script>
import { ref, inject, mounted, onBeforeUnmount } from "vue";
import EventBus from "./composables/event";

const receivedName = ref(null);

const handleFormSubmit = (payload) => {
  receivedName.value = payload;
};

export default {
  setup() {
    return {
      receivedName,
    };
  },

  mounted() {
    EventBus.on("form-submitted", handleFormSubmit);
  },

  onBeforeUnmount() {
    EventBus.off("form-submitted");
  },
};
</script>
```

### Conclusion

An event bus is a design pattern that allows components to communicate with each other without being directly connected. In the publish/subscribe (pubsub) pattern, a publisher (event emitter) triggers an event and any number of subscribers (event listeners) can respond to it.

In Vue 2, the event bus can be implemented by creating a new Vue instance and defining it on the prototype, allowing it to be accessed from any component. On the contrary in Vue 3 we have to write our own pub-sub composable for this or use a package. 

The event bus can be a useful tool for communication between decoupled components, but it is generally not recommended for use in the long term due to the potential for maintenance issues.