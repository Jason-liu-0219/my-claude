---
name: vue3
description: Vue framework for UIs
---

# Vue3 Skill

Vue.js 3 - The Progressive JavaScript Framework for building user interfaces. This skill synthesizes knowledge from official Vue.js documentation to provide comprehensive guidance on Vue 3 development patterns, APIs, and best practices.

## When to Use This Skill

This skill should be triggered when:

### Component Development
- Creating Vue 3 Single-File Components (SFCs)
- Working with `<script setup>` syntax
- Defining props, emits, and component options
- Implementing component v-model bindings
- Managing slots and template refs

### Reactivity & State
- Using `ref()`, `reactive()`, `computed()` for state management
- Implementing watchers with `watch()` and `watchEffect()`
- Understanding Vue's reactivity system
- Working with `toValue()`, `toRef()`, `unref()` utilities

### Composition API
- Writing composable functions (hooks)
- Using lifecycle hooks (`onMounted`, `onUnmounted`, etc.)
- Implementing dependency injection with `provide`/`inject`
- Organizing code with the Composition API

### Template Syntax
- Using Vue directives (`v-if`, `v-for`, `v-bind`, `v-on`, `v-model`)
- Working with dynamic arguments and modifiers
- Handling events and form inputs
- Class and style bindings

### TypeScript Integration
- Type-safe props and emits with TypeScript
- Generic components
- Type inference in templates

### Best Practices
- Accessibility (a11y) implementation
- Performance optimization
- Security considerations
- Style guide compliance

## Key Concepts

### Composition API vs Options API
Vue 3 supports both APIs. The **Composition API** (with `<script setup>`) is recommended for:
- Better TypeScript support
- More flexible code organization
- Improved code reuse through composables
- Better IDE type-inference

### Reactivity Fundamentals
- **`ref()`**: For primitive values, accessed via `.value`
- **`reactive()`**: For objects, accessed directly
- **`computed()`**: For derived values with caching
- Refs are automatically unwrapped in templates

### Single-File Components (SFCs)
The `.vue` file format combines template, script, and styles in one file. The `<script setup>` syntax provides compile-time optimizations and cleaner code.

## Quick Reference

### Basic Component with Script Setup
*From official docs - recommended syntax for SFCs*

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}
</script>

<template>
  <button @click="increment">{{ count }}</button>
</template>
```

### Defining Props and Emits
*From official docs - using compiler macros*

```vue
<script setup>
const props = defineProps({
  foo: String
})

const emit = defineEmits(['change', 'delete'])
</script>
```

### Type-Safe Props with TypeScript
*From official docs - type-only declaration*

```ts
const props = defineProps<{
  foo: string
  bar?: number
}>()

const emit = defineEmits<{
  change: [id: number]
  update: [value: string]
}>()
```

### Props with Default Values (3.5+)
*From official docs - reactive props destructure*

```ts
interface Props {
  msg?: string
  labels?: string[]
}

const { msg = 'hello', labels = ['one', 'two'] } = defineProps<Props>()
```

### Two-Way Binding with defineModel (3.4+)
*From official docs - simplified v-model implementation*

```vue
<script setup>
// declares "modelValue" prop, consumed via v-model
const model = defineModel()

// OR: with options
const model = defineModel({ type: String })

// Named model: consumed via v-model:count
const count = defineModel('count', { type: Number, default: 0 })
</script>
```

### Exposing Component Methods
*From official docs - for template refs*

```vue
<script setup>
import { ref } from 'vue'

const a = 1
const b = ref(2)

defineExpose({
  a,
  b
})
</script>
```

### Custom Directive in Script Setup
*From official docs - local directives*

```vue
<script setup>
const vMyDirective = {
  beforeMount: (el) => {
    // do something with the element
  }
}
</script>

<template>
  <h1 v-my-directive>This is a Heading</h1>
</template>
```

### Creating a Composable
*From official docs - reusable stateful logic*

```js
// mouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}
```

Using the composable:

```vue
<script setup>
import { useMouse } from './mouse.js'

const { x, y } = useMouse()
</script>

<template>Mouse position: {{ x }}, {{ y }}</template>
```

### Async Data Fetching Composable
*From official docs - handling loading/error states*

```js
// fetch.js
import { ref, watchEffect, toValue } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const error = ref(null)

  watchEffect(() => {
    data.value = null
    error.value = null

    fetch(toValue(url))
      .then((res) => res.json())
      .then((json) => (data.value = json))
      .catch((err) => (error.value = err))
  })

  return { data, error }
}
```

### Template Syntax - Text & HTML
*From official docs - data binding*

```vue
<script setup>
const msg = 'Hello!'
const rawHtml = '<span style="color: red">Red text</span>'
</script>

<template>
  <!-- Text interpolation -->
  <p>{{ msg }}</p>

  <!-- Raw HTML (use with caution!) -->
  <span v-html="rawHtml"></span>
</template>
```

### Attribute Bindings
*From official docs - v-bind shorthand*

```vue
<template>
  <!-- Full syntax -->
  <div v-bind:id="dynamicId"></div>

  <!-- Shorthand -->
  <div :id="dynamicId"></div>

  <!-- Same-name shorthand (3.4+) -->
  <div :id></div>

  <!-- Boolean attribute -->
  <button :disabled="isDisabled">Submit</button>

  <!-- Multiple attributes -->
  <div v-bind="objectOfAttrs"></div>
</template>
```

### Event Handling
*From official docs - v-on shorthand*

```vue
<template>
  <!-- Full syntax -->
  <button v-on:click="handleClick">Click</button>

  <!-- Shorthand -->
  <button @click="handleClick">Click</button>

  <!-- With modifiers -->
  <form @submit.prevent="onSubmit">...</form>

  <!-- Dynamic event name -->
  <button @[eventName]="handler">Click</button>
</template>
```

### Watch with Route Changes
*From official docs - accessibility pattern*

```vue
<script setup>
import { ref, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const backToTop = ref()

watch(
  () => route.path,
  () => {
    backToTop.value.focus()
  }
)
</script>
```

### Generic Components (TypeScript)
*From official docs - type parameters*

```vue
<script setup lang="ts" generic="T extends string | number">
defineProps<{
  items: T[]
  selected: T
}>()
</script>
```

## Reference Files

This skill includes comprehensive documentation in `references/`:

### llms-full.md
**Source:** Official Vue.js documentation
**Confidence:** High
**Contents:**
- Complete `<script setup>` API documentation
- All compiler macros (`defineProps`, `defineEmits`, `defineModel`, `defineExpose`, `defineOptions`, `defineSlots`)
- Composition API reference
- Reactivity API (ref, reactive, computed, watch)
- Built-in directives and components
- TypeScript integration guides
- Accessibility best practices
- Animation techniques
- Style guide rules

### llms.md
**Source:** Official Vue.js documentation
**Confidence:** High
**Contents:**
- Table of contents for all Vue.js documentation
- Quick navigation to specific topics
- Links to guides, API reference, and best practices

### other.md
**Source:** Official Vue.js documentation
**Confidence:** Medium
**Contents:**
- Template syntax fundamentals
- Text interpolation and raw HTML
- Attribute bindings (v-bind)
- JavaScript expressions in templates
- Directive syntax overview

### index.md
**Source:** Documentation index
**Confidence:** Medium
**Contents:**
- Category overview
- File organization

## Working with This Skill

### For Beginners
1. Start with the **Template Syntax** section in `other.md` to understand Vue's declarative rendering
2. Learn `<script setup>` basics from `llms-full.md` - this is the recommended approach
3. Practice with the Quick Reference examples above
4. Focus on `ref()` and `reactive()` for state management

### For Intermediate Developers
1. Explore composables for code reuse patterns
2. Learn TypeScript integration for type-safe components
3. Study the Composition API lifecycle hooks
4. Review accessibility best practices

### For Advanced Users
1. Dive into advanced reactivity patterns (`toValue`, `customRef`, etc.)
2. Explore generic components with TypeScript
3. Study performance optimization techniques
4. Review the style guide rules for consistency

### Navigation Tips
- Use `llms.md` as a table of contents to find specific topics
- `llms-full.md` contains the most comprehensive information
- Search for specific APIs or patterns by name
- Code examples are marked with language tags for syntax highlighting

## Common Patterns

### Component Communication
| Pattern | Use Case |
|---------|----------|
| Props | Parent to child data |
| Emits | Child to parent events |
| v-model | Two-way binding |
| Provide/Inject | Deep component trees |
| Composables | Shared stateful logic |

### Reactivity Choices
| API | Best For |
|-----|----------|
| `ref()` | Primitives, template refs |
| `reactive()` | Objects, complex state |
| `computed()` | Derived values |
| `watch()` | Side effects on changes |
| `watchEffect()` | Auto-tracked side effects |

### Script Setup Macros
| Macro | Purpose |
|-------|---------|
| `defineProps()` | Declare component props |
| `defineEmits()` | Declare emitted events |
| `defineModel()` | Declare v-model bindings (3.4+) |
| `defineExpose()` | Expose public methods |
| `defineOptions()` | Set component options (3.3+) |
| `defineSlots()` | Type slot props (3.3+) |

## Security Considerations

From the official documentation:

- **v-html Warning**: Dynamically rendering arbitrary HTML can lead to XSS vulnerabilities. Only use `v-html` on trusted content and **never** on user-provided content.
- **Template Expressions**: Are sandboxed and only have access to a restricted list of globals.
- Always validate and sanitize user input before rendering.

## Resources

### references/
Organized documentation extracted from official Vue.js sources:
- Detailed explanations with context
- Code examples with proper language annotations
- Links to original documentation
- Structured table of contents

### Official Links
- [Vue.js Documentation](https://vuejs.org/)
- [Vue.js Playground](https://play.vuejs.org/)
- [Vue.js GitHub](https://github.com/vuejs/core)

### scripts/
Add helper scripts here for common automation tasks.

### assets/
Add templates, boilerplate, or example projects here.

## Version Notes

This skill covers Vue 3.x features, with specific version requirements noted:
- **3.3+**: `defineOptions()`, `defineSlots()`, complex type imports in props
- **3.4+**: `defineModel()`, same-name shorthand for v-bind
- **3.5+**: Reactive Props Destructure (enabled by default)

## Notes

- This skill was synthesized from official Vue.js documentation
- All code examples are from official sources with high confidence
- Reference files preserve structure and examples from source docs
- Code examples include language detection for syntax highlighting
- Composable naming convention: start with "use" (e.g., `useMouse`, `useFetch`)

## Updating

To refresh this skill with updated documentation:
1. Re-run the scraper with the same configuration
2. The skill will be rebuilt with the latest information
3. Check Vue.js release notes for new features
