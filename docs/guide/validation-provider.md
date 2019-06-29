# Validation Provider

The `ValidationProvider` component is a regular component that wraps your inputs and provides validation state using [scoped slots](https://vuejs.org/v2/guide/components-slots.html#Scoped-Slots).

:::tip
The slots syntax has been changed in Vue 2.6, the following examples use the new `v-slot` syntax instead of the deprecated `slot-scope`, but it is still supported and you can use it. However `v-slot` have different semantics, consult the Vue docs for more information.
:::

Using the ValidationProvider offers isolated scope for each field validation state, and does not inject/mutate anything outside its slot. You can import it and use whenever you need it. Using the validation context will allow you to apply classes, flags and pass state to your template.

Here is a quick example:

```vue{2,5,9,13}
<template>
  <ValidationProvider rules="required" v-slot="{ errors }">
    <input v-model="value" type="text">
    <span id="error">{{ errors[0] }}</span>
  </ValidationProvider>
</template>

<script>
import { ValidationProvider } from 'vee-validate';

export default {
  components: {
    ValidationProvider
  }
};
</script>
```

It also works for custom components and solves the issue of creating components with built-in validation.

:::tip
  The fields being validated __must have__ a `v-model` so the component can correctly identify the element/component being validated. Otherwise you need to handle the validation manually.
:::

## Rendering

By default, ValidationProvider renders a `span`, Consider the following example where the highlighted represent the render output of the Provider component.

```vue{7,10}
  <ValidationProvider rules="required" v-slot="{ errors }">
    <input v-model="value" type="text">
    <span>{{ errors[0] }}</span>
  </ValidationProvider>

  <!-- HTML Output -->
  <span>
    <input type="text">
    <span>ERROR_MSG_PLACEHOLDER</span>
  </span>
```

The default the rendered **tag** can be changed using the provider's `tag` prop.

```vue{2,10,16,19}
  <!-- Multiple Child nodes using templates -->
  <ValidationProvider rules="required" tag="div">
    <template v-slot="{ errors }">
      <input v-model="value" type="text">
      <span>{{ errors[0] }}</span>
    </template>
  </ValidationProvider>

  <!-- Multiple Child nodes directly -->
  <ValidationProvider rules="required" v-slot="{ errors }" tag="div">
    <input v-model="value" type="text">
    <span>{{ errors[0] }}</span>
  </ValidationProvider>

  <!-- Both have the same HTML Output -->
  <div>
    <input type="text">
    <span>ERROR_MSG_PLACEHOLDER</span>
  </div>
```

### Forcing Renderless

Sometimes it is unsuitable for a Provider component in principle to render anything extra, because of limitations in the Vue rendering engine we cannot have multiple root nodes which limits the design choice to move away from renderless at the moment, in Vue 3.x it this may change with fragments.

A `slim` prop can be used to force the component to be renderless, by default it is set to `false`.

```vue{2,5,9,12}
<ValidationProvider rules="required" v-slot="{ errors }" slim>
  <div>
    <input v-model="value" type="text">
    <span>{{ errors[0] }}</span>
  </div>
</ValidationProvider>

<!-- HTML Output, NO OUTER SPANS -->
<div>
  <input type="text">
  <span>ERROR_MSG_PLACEHOLDER</span>
</div>
```

Note that **only the first child** will be rendered when `slim` is used, any other nodes will be dropped as you cannot have multiple root nodes in a renderless component. Be mindful of that when using the `slim` prop.

```vue{2,7}
<ValidationProvider rules="required" v-slot="{ errors }" slim>
  <input v-model="value" type="text">
  <span>{{ errors[0] }}</span>
</ValidationProvider>

<!-- Only input is rendered, the span is dropped -->
<input type="text">
```

## Scoped Slot Data

The object passed down to the slot scope is called the __validation context__. It has the following properties:

| Name    | Type                       |  Description |
|:--------|:--------------------------:|:--------------------------------------------------------------------|
| errors  | `string[]`                 | The list of error messages.                                         |
| failedRules | `[x: string]: string`  | A map object of failed rules with (rule, message) as a (key, value) |
| valid   | `boolean`                  | The current validation state.                                       |
| flags   | `{ [x: string]: boolean }` | The flags map object state.                                         |
| aria    | `{ [x: string]: string }`  | Map object of aria attributes for accessibility.                    |
| classes | `{ [x: string]: boolean }` | Map object of the classes configured based on the validation state. |
| validate| `(e: any) => Promise`      | A function that is used as an event handler to trigger validation. Useful for fields that do not use v-model. |
| reset   | `() => void`               | A function that resets the validation state on the provider.        |

Since slot scopes can take advantage of ES6 destructing, you can opt-in for any of those properties and pass down to your slot template as you see fit. The example above only needed the `errors` array.

## Examples

The previous quick sample validates simple HTML inputs, lets take this up a notch and validate popular 3rd party components like [Vuetify's TextInput](https://vuetifyjs.com/en/components/text-fields).

### Manual Validation

Triggering validation on any of the providers is simple, but it is opt-in. Meaning you need to explicitly call the validation on the provider instance. Using [refs](https://vuejs.org/v2/api/#ref) and the [public method](#methods) `validate` makes it straight forward.

```vue{3,8,18}
<template>
  <div>
    <ValidationProvider rules="required" ref="myinput" v-slot="{ errors }">
      <input v-model="value" type="text">
      <span>{{ errors[0] }}</span>
    </ValidationProvider>

    <button @click="validateField()" >Submit</button>
  </div>
</template>

<script>
export default {
  // ...
  methods: {
    validateField () {
      // Validate the field
      return provider.validate();
    }
  },
  // ..
};
</script>
```

If you only plan to trigger manual validation using the UI, you can use the `validate` handler on the v-slot data to trigger validation without having to use `refs`.

```vue{1}
<ValidationProvider rules="required" v-slot="{ validate, errors }">
  <div>
    <input type="text" @input="validate">
    <p id="error">{{ errors[0] }}</p>
  </div>
</ValidationProvider>
```

Note that the `validate` function can handle DOM events and value validation as well. You can use the `$event` in the Vue template to reference the event arg that is emitted with the event if your handlers are more complex.

```vue
<ValidationProvider rules="required" v-slot="{ validate, errors }">
  <div>
    <input type="text" @input="syncVuex($event.target.value) || validate($event)">
    <p id="error">{{ errors[0] }}</p>
  </div>
</ValidationProvider>
```

:::tip
  Using the same approach you can reset validation state for the provider using the public method `reset()` and the slot scope method of the same name.
:::

### Model-less Validation

While `v-model` is generally required when using the ValidationProvider component, some inputs like `file` do not benefit from having `v-model`. Instead, you can use the manual `validate` method to avoid having to use `v-model` in this instance.

```vue
<ValidationProvider rules="required|image" v-slot="{ validate, errors }">
  <div>
    <input type="file" @change="handleFileChange($event) || validate($event)">
    <p id="error">{{ errors[0] }}</p>
  </div>
</ValidationProvider>
```

### Input Groups (Checkbox/Radio)

Like radio inputs and (sometimes) check boxes, some inputs behave as a single input entity. You can wrap a whole group of inputs __given that they have the same `v-model`__ in a single ValidationProvider component. You can group as many inputs as you want inside the ValidationProvider component.

```vue
<ValidationProvider rules="required" v-slot="{ errors }">
  <div>
    <input type="radio" v-model="drink" value="">
    <input type="radio" v-model="drink" value="coffee">
    <input type="radio" v-model="drink" value="coke">
  </div>
</ValidationProvider>
```

### Cross-Field Validation

When using cross-field rules like the `confirmed` rule the target field must have a corresponding `vid` can be either a number or a string.

```vue{2,9}
<ValidationProvider
  rules="required|confirmed:confirm"
  v-slot="{ errors }"
>
  <input v-model="password" type="text">
  <span>{{ errors[0] }}</span>
</ValidationProvider>

<ValidationProvider
  vid="confirm"
  rules="required"
  v-slot="{ errors }"
>
  <input v-model="confirmation" type="text">
  <span>{{ errors[0] }}</span>
</ValidationProvider>
```

[You can see providers in action here](/examples/validation-providers.md)

## Refactoring Validation Providers

While the ValidationProvider has its advantages, it is verbose and can be very annoying when creating large forms. There are a couple of ways to address this issue.

### Higher-Order Components

A common pattern is to use higher-order components to produce new components with slightly different behavior. This is similar to creating a wrapper or a mixin for our component, except it uses props/events to communicate state.

VeeValidate exposes a `withValidation` function takes a component as an argument and creates a new one with the validation behavior enabled. Let's create a `TextInput` component then validate it using this method:

This is the `TextInput` Component:

```vue
<template>
  <div>
    <input type="text" @input="$emit('input', $event.target.value)" :value="value">
    <span v-show="error">{{ error }}</span>
  </div>
<template>

<script>
export default {
  name: 'TextField',
  props: ['value', 'error']
};
</script>
```

Now this component does not have validation built-in, but it does have the UI to do so which is common in design systems and input components.

The next step is to use the `withValidation` function to create a version of it but with validation enabled.

```js
import { withValidation } from 'vee-validate';
import TextField from './components/TextField';

const TextFieldWithValidation = withValidation(TextField, ctx => ({
  'error': ctx.errors[0]
}));
```

The `TextFieldWithValidation` is an **enhanced component** that we can now use anywhere. But a small problem appears, **how are you going to specify `rules`?**, what does this new component accept as props?

That the second parameter is a function that transforms the validation scoped slot to props object to be passed to the wrapped component. In this case we want to pass the first item in the `errors` array as the `error` prop to the underlying `TextField` component.

This is the full contents of the `ctx` object:

```ts
interface ValidationContext {
  errors: string[];
  flags: ValidationFlags;
  classes: ValidationClassMap;
  valid: boolean;
  failedRules: { [k: string]: string };
  reset: () => void;
  validate: (evtOrNewValue: Event | any) => Promise<ValidationResult>;
  aria: {
    'aria-invalid': 'true' | 'false';
    'aria-required': 'true' | 'false';
  };
}
```

With this approach the last example becomes:

```vue
<TextFieldWithValidation rules="required|confirmed:confirm" v-model="password" />

<TextFieldWithValidation vid="confirm" rules="required" v-model="password" />
```

:::danger
This approach has some cons, for example if the wrapped component accepts props that have the same name as the `ValidationProvider` component. while it will receive these props, they may be of different types, which could lead to serious issues. The problem with HOCs are that you need to be aware of the underlying component's implementation. This can be problematic when working with 3rd party components.
:::

### Wrapping Components Manually

Instead we can wrap the field component with the ValidationProvider in a new component. This is easier and more flexible, and avoids some of the potential problems with Higher-Order components.

Consider this new `TextFieldWithValidation` component.

```vue
<template>
  <ValidationProvider :rules="rules" v-slot="{ errors }">
    <TextField v-model="innerValue" :error="errors[0]" />
  </ValidationProvider>
</template>

<script>
import { ValidationProvider } from 'vee-validate';
import TextField from './components/TextField';

export default {
  name: 'TextFieldWithValidation',
  props: {
    rules: [String], // we need this
    value: null
  },
  components: {
    ValidationProvider,
    TextField
  },
  data: () => ({
    innerValue: null
  }),
  watch: {
    innerValue (val) {
      this.$emit('input', val);
    }
  }
};
</script>

```

Ideally you would pass the props you need to either the `ValidationProvider` or the `TextField` being validated, with this approach solves the verbosity problem while preserving the simple scoped slots API. It also allows you to distribute props without the issues of having a conflict, unlike HOC.

Using either of these approaches is at your preference.

## Adding Errors Manually

You may want to add manual errors to a field, such cases like pre-filling initially due to a server response, or an async request. You can do this using `refs` and the `applyResult` method.

```vue{3,19,20,21,22,23}
<template>
  <div id="app">
    <ValidationProvider ref="provider" rules="required" v-slot="{ errors }">
      <input v-model="model" type="text">
      <div v-if="errors" v-text="errors[0]"></div>
    </ValidationProvider>
  </div>
</template>

<script>
import { ValidationProvider } from "vee-validate";

export default {
  components: {
    ValidationProvider
  },
  mounted() {
    // all those properties are required.
    this.$refs.provider.applyResult({
      errors: ["this is a backend error"], // array of string errors
      valid: false, // boolean state
      failedRules: {} // should be empty since this is a manual error.
    });
  }
};
</script>
```

:::tip Typescript and $refs
If you are using TypeScript you may face issues with `$refs` not giving you the correct typings, you can solve that by defining them as `ValidationProvider` instances:

```ts{3}
export default class App extends Vue {
  $refs!: {
    provider: InstanceType<typeof ValidationProvider>;
  }
}
```

```ts{3,4,5}
// All properties are required.
this.$refs.provider.applyResult({
  errors: ["this is a backend error"],
  valid: false,
  failedRules: {}
});
```

:::

## Reference

Below is the reference of the ValidationProvider public API.

### Props

All the following props are optional.

|Prop       | Type      | Default Value         | Description                                                                  |
|-----------|-----------|-----------------------|------------------------------------------------------------------------------|
| rules     | `string`  | `undefined`           | The validation rules.                                                        |
| vid       | `string`  | auto increment number | Identifier used for target/cross-field based rules.                          |
| immediate | `boolean` | `false`               | If the field should be validated immediately after render (initially).       |
| events    | `string[]`| `['input']`           | deprecated: check [interaction modes](../interaction.md)                      |
| name      | `string`  | `undefined`           | A string that will be used to replace `{field}` in error messages and for [custom error messages](/guide/messages.md#field-specific-custom-messages). |
| bails     | `boolean` | `true`                | If true, the validation will stop on the first failing rule.                 |
| debounce  | `number`  | `0`                   | Debounces the validation for the specified amount of milliseconds.           |
| tag       | `string`  | `span`                | The default tag to [render](#rendering). |
| persist   | `boolean` | `false`               | If true, the provider will keep its errors across mounted/destroyed lifecycles |

### Methods

Those are the only methods meant for public usage, other methods that may exist on the ValidationProvider are strictly internal.

|Method       | Args    | Return Value                | Description                                                                                                 |
|-------------|:-------:|:---------------------------:|-------------------------------------------------------------------------------------------------------------|
| validate    | `value?: any`  | `Promise<ValidationResult>` | Runs a validation of the current value against the rules defined. If a value is provided, it is used as the current value and validates it. |
| validateSilent | `void`  | `Promise<ValidationResult>` | Runs a validation of the current value against the rules defined. __does not mutate the validation state.__  |
| applyResult | `ValidationResult` | `void`           | Takes a __validation result__ object and applies it on the current state.                                   |
| reset       | `void`  | `void`                      | Resets validation state.                                                                                    |
| setFlags    | Object | void | Updates the field flag states with an object, the object properties should be the flag names and the values should be boolean values. |

### Events

The validation provider does not emit any events at this time.

---

What now remains, is how to validate forms, the `ValidationObserver` is the other component that completes this library.