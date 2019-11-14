- Start Date: 2019-11-14
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Remove support for the [inline-template feature](https://vuejs.org/v2/guide/components-edge-cases.html#Inline-Templates).

# Motivation

`inline-template` was originally included in Vue to address the cases where Vue is used to progressively enhance a largely server-rendered application. It allows users to define the template of a child component directly inside a parent's template.

The biggest issue with `inline-template` is that it makes template scoping very inconsistent. Without `inline-template`, a simple rule of thumb is that every variable appearing inside a template is either provided by the owner component, or by a directive that explicitly introduces scope variables (e.g. `v-for` and `v-slot`). `inline-template` breaks that assumption by mixing multiple scoping contexts in the same template:

``` html
<div>
  {{ parentMsg }}
  <child-comp inline-template>
    {{ parentMsg }}
  </child-comp>
</div>
```

In a standard component expecting slots, `{{ parentMsg }}` would work intuitively inside the slot content. However with `inline-template`, that is no longer the case. Similarly components with `v-for` + `inline-template` won't work as expected either:

``` html
<child-comp inline-template v-for="item in list">
  {{ item.msg }}
</child-comp>
```

Here the inner template actually has no access to the iterated `item`. It's pointing to `this.item` on the child component instead.

# Adoption strategy

Most of the use cases for `inline-template` assumes a no-build-tool setup, where all templates are written directly inside the HTML page. The most straightforward workaround in such cases is using `<script>` with an alternative type:

``` html
<script type="text/html" id="my-comp-template">
  <div>
    {{ hello }}
  </div>
</script>
```

And in the component, target the template using a selector:

``` js
const MyComp = {
  template: '#my-comp-template',
  // ...
}
```

This doesn't require any build setup, works in all browsers, is not subject to in-DOM HTML parsing caveats (e.g. you can use camelCase prop names), and provides proper syntax highlighting in most IDEs. In a traditional server-side framework, these templates can be split out into server template partials (included into the main HTML template) for better maintainability.
