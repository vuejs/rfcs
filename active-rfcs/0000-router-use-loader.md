- Start Date: 2022-07-14
- Target Major Version: Vue 3, Vue Router 4
- Reference Issues:
- Implementation PR:

# Summary

Provide a set of functions `defineLoader()` and `useLoader()` to standardize and improve data fetching:

- Automatically integrate fetching to the navigation cycle
- Automatically rerun when used params changes (avoid unnecessary fetches)
- Provide control over loading/error states

# Basic example

If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable.

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from '@vue-router'

export const loader = defineLoader(async (route) => {
  const user = await getUserById(route.params.id)
  // ...
  return { user }
})

// Optional: define other component options
export default defineComponent({
  name: 'custom-name',
  inheritAttrs: false
})
</script>

<script lang="ts" setup>
import { useLoader } from '@vue-router'

const { user, isLoading, error } = useLoader()
// user is always present, isLoading changes when going from '/users/2' to '/users/3'
</script>
```

# Motivation

There are currently too many ways of handling data fetching with vue-router and all of them have problems:

- With navigation guards:
  - using `meta`: complex to setup even for simple cases, too low level for such a simple case
  - using `onBeforeRouteUpdate()`: missing the enter part
  - `beforeRouteEnter()`: non typed and non-ergonomic API with `next()`, requires a data store (pinia, vuex, apollo, etc)
  - using a watcher on `route.params...`: component renders without the data (doesn't work with SSR)

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Vue to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## TypeScript

Types are automatically generated for the routes by [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) and can be referenced with the name of each route to hint `defineLoader()` and `useLoader()` the possible values of the current types:

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from '@vue-router'

export const loader = defineLoader('/users/[id]', async (route) => {
  //                                ^ autocompleted
  const user = await getUserById(route.params.id)
  //                                          ^ typesafe
  // ...
  return { user }
})
</script>

<script lang="ts" setup>
import { useLoader } from '@vue-router'

const { user, isLoading, error } = useLoader('/users/[id]')
//                                            ^ autocompleted
// same as
const { user, isLoading, error } = useLoader<'/users/[id]'>()
</script>
```

The arguments can be removed during the compilation step in production mode since they are only used for types and are actually ignored at runtime.

# Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Vue
- integration of this feature with other existing and planned features
- cost of migrating existing Vue applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

- Adding a new `<script loader>` similar to `<script setup>`:

```vue
<script lang="ts" loader>
import { getUserById } from '@/api/users'
import { useRoute } from '@vue-router' // could be automatically imported

const route = useRoute()
// any variable created here is available in useLoader()
const user = await getUserById(route.params.id)
</script>

<script lang="ts" setup>
import { useLoader } from '@vue-router'

const { user, isLoading, error } = useLoader()
</script>
```

Is exposing every variable a good idea?

- Using Suspense to natively handle `await` within `setup()`. [See other RFC](#TODO).

# Adoption strategy

Introduce this as part of [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to test it first and make it part of the router later on.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
