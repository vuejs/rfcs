- Start Date: 2021-02-02
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/3131
- Implementation PR: (leave this empty)

# Summary

Add a `gravity` prop to `<transition-group>` that changes the point on the `ClientRect` used to calculate the translation delta for `-move` transitions. This will enable smooth `-move` transitions on lists whose items stack toward the bottom or to the right, where currently, if the item changes in size, it jumps unintuitively before beginning the `-move` transition.

# Basic example

Imagine in this example, each `.chat-message` stacks toward the bottom of `.chat-message-group`.

``` html
<transition-group 
    tag="div" 
    name="chat-message" 
    gravity="bottom" 
    class="chat-message-group"
>
    <div 
        class="chat-message" 
        v-for="item in items" 
        :key="item.key"
    >
        <p class="chat-message-user">Example user name</p>
        <div class="some-element-that-is-toggled"/>
        <p class="chat-message-user">{{ item.text }}</p>
    </div>
</transition-group>
```

# Motivation

`<transition-group>` works well today for lists whose items stack to the left or the top, as most lists do on the web. However, for lists where items stack toward the bottom (like in some chat applications), or toward the right (like horizontal lists in right-to-left languages), elements that change size jump unintuitively before transitioning. 

This happens because when the transformation between its previous and new position is calculated, `<transition-group>` simply measures the item's position from the top left. In order to correctly represent where the element "is," you must consider which direction gravity is facing. For example, if the items stack toward the bottom, the element's position should be represented by a point on the bottom edge of its bounding rect.

It's a subtle problem, perhaps one that most wouldn't notice, but it has an impact on the "readability" of a transition in these cases. Some simple logic might be added to determine the reference point for measuring the item's position based on a `gravity` prop. The change seems simple enough to justify a small gain for developers seeking polished list transitions.

[Here is a codesandbox](https://codesandbox.io/s/transition-group-bottom-anchored-example-nv3bh?file=/src/App.vue) demonstrating the current and desired behavior.

# Detailed design

## New Prop: `gravity`
``` ts
// TransitionGroup.ts:42
const TransitionGroupImpl = {
    //...

    props: /*#__PURE__*/ extend({}, TransitionPropsValidators, {
        tag: String,
        moveClass: String,
        gravity: {
            type: String as PropType<"top" | "top-right" | "right" | "bottom-right" | "bottom" | "bottom-left" | "left" | "top-left">,
            default: "top-left"
        }
    }),

    //...
}
```

### Options
The possible options are:
- `"left"`: Measure translation delta using the midpoint of the rect's left side.
- `"top-left"`: The default. Measure translation delta using the top left of the rect.
- `"top"`: Measure translation delta using the midpoint of the rect's top side.
- `"top-right"`: Measure translation delta using the top right of the rect.
- `"right"`: Measure translation delta using the midpoint of the rect's right side.
- `"bottom-right"`: Measure translation delta using the bottom right of the rect.
- `"bottom"`: Measure translation delta using the midpoint of the rect's bottom side.
- `"bottom-left"`: Measure translation delta using the bottom left of the rect.

### Relevant Changes in the Source Code
``` ts
// TransitionGroup.ts

// changed to represent x and y abstractly.
interface Position {
  x: number;
  y: number;
}

//...

// instead of masking ClientRect with the Position interface...
positionMap.set(child, getPosition(child, props.gravity));

//...

// something like...  
function getPosition(child: VNode, gravity: string): Position {
  const rect = (child.el as Element).getBoundingClientRect();
  let x: number, y: number;
  switch (gravity) {
    case "left":
      x = rect.left;
      y = rect.top + rect.height / 2;
      break;
    case "bottom-left":
      x = rect.left;
      y = rect.bottom;
      break;
    case "bottom":
      x = rect.left + rect.width / 2;
      y = rect.bottom;
      break;
    case "bottom-right":
      x = rect.right;
      y = rect.bottom;
      break;
    case "right":
      x = rect.right;
      y = rect.top + rect.height / 2;
      break;
    case "top-right":
      x = rect.right;
      y = rect.top;
      break;
    case "top":
      x = rect.left + rect.width / 2;
      y = rect.top;
    case "top-left":
    default:
      x = rect.left;
      y = rect.top;
      break;
  }
  return { x, y };
}

//...

// here simply replace .left and .top with .x and .y
function applyTranslation(c: VNode): VNode | undefined {
  const oldPos = positionMap.get(c)!;
  const newPos = newPositionMap.get(c)!;
  const dx = oldPos.x - newPos.x;
  const dy = oldPos.y - newPos.y;
  if (dx || dy) {
    const s = (c.el as HTMLElement).style;
    s.transform = s.webkitTransform = `translate(${dx}px,${dy}px)`;
    s.transitionDuration = "0s";
    return c;
  }
}
```

## Use Cases
[See this codesandbox](https://codesandbox.io/s/transition-group-bottom-anchored-example-nv3bh?file=/src/App.vue) demonstrating an implementation of the proposed syntax for these use cases.

### Chat-style applications where messages stack toward the bottom.
``` html
<transition-group 
    tag="div" 
    name="chat-message" 
    gravity="bottom" 
    class="chat-message-group"
>
    <div 
        class="chat-message" 
        v-for="item in items" 
        :key="item.key"
    >
        <p class="chat-message-user">Example user name</p>
        <div class="some-element-that-is-toggled"/>
        <p class="chat-message-user">{{ item.text }}</p>
    </div>
</transition-group>
```

### Horizontal lists in right-to-left languages

``` html
<template>
    <transition-group tag="ul" class="vegetable-list" transition="horizontal-list" gravity="top-right">
        <div v-for="vegetable in vegetables" class="vegetable" :key="vegetable.key">{{ vegetable.name }}</div>
    </transition-group>
</template>

<script>
export default {
    name: "VegetableList",
    //...
    data () {
        return {
            vegetables: [
                {key: 0, name: "بروكلي"},
                {key: 1, name: "بنجر"},
                {key: 2, name: "ثوم"},
                {key: 3, name: "شمر"},
            ]
        }
    },
    //...
}
</script>
```

# Drawbacks

- The problem is somewhat difficult to explain. The issue is subtle enough that, unless you've witnessed it, you might wonder if you're missing something if you encountered this property in the docs.
- It involves rethinking the private `Position` interface. Simply masking ClientRect would not be enough.
- Workarounds
    - One could "clone-and-own" the `TransitionGroup` component, modifying it to measure from the right or bottom as appropriate.
    - One could reimplement the F.L.I.P. animation using the programmatic transition api.
    - Someone in the community could create and publish a very thin `BottomTransitionGroup` component.

# Alternatives

There is no elegant workaround for the use cases mentioned. One might "clone-and-own" `TransitionGroup.ts`, modifying `.left` and `.top` to `.right` and `.bottom` or whatever combination is appropriate for their use case, but I suspect most users will be uncomfortable making that kind of modification. Doing so also increases the bundle size.

Alternatively, you may consider more magical approaches like checking the `writing-mode` or `flex-direction` or `grid-auto-flow` style properties of the `transition-group` element.

# Adoption strategy

Making this change will be transparent for almost all users, as the default behavior without specifying this property would be identical to how it works today.

# Unresolved questions

- An alternative name for this prop might be `anchor`. But it seems less intuitive since it implies something about the subject `transition-group`'s children instead of itself.
- Are there any use cases for a `center` option?
- Is it even necessary to have that many options? Would `"top-left"`, `"bottom-right"`, `"top-right"`, and `"bottom-right"` be sufficient for all use cases? Are there better names for these gravity directions. `"normal"` or `"reverse"` maybe?
    - Even if those are the only use cases, does omitting `"top"`, `"bottom"`, `"left"` and `"right"` make the property less intuitive?
- How might this apply to two-dimensional layouts like `display: grid`?
    - Is there a use case here for `"center"`?
- Unclear if it might interfere with certain third-party libraries (such as animation / transition libraries). Breaking changes seem unlikely, since the default behavior is identical.
- Are there "magical" approaches, like checking the `writing-mode` or `flex-direction` or `grid-auto-flow` style properties of the element?
    - Scrollable lists that stack toward the bottom are often implemented as a normal column-flowing block container (the `transition-group`) wrapped in a column-reverse-scrolling flex wrapper. In this case it might be impractical to infer which direction gravity is pointing?
- Hypothetically, other use cases might be "tags", "presence indicators", "live chat overlays" in videos.
