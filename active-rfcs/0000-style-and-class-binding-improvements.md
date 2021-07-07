- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

* Introduce named argument class and style bindings
* Add `prop` and unit modifiers for argument style bindings

# Basic example

## Class

```html
<!-- Before -->
<input :class="[{ active: isActive }, dynamicClass]">

<!-- After -->
<input :class:active="isActive" :class[dynamicClass]>
```

## Style
```html
<!-- Before -->
<input :style="{ '--custom-property': propertyValue + 'px' }">

<!-- After -->
<input :style:custom-property.prop.px="propertyValue">
```

# Motivation

## Overcomplicated complex class bindings

Vue has inherited an AngularJS style of handling complex style and class bindings via an object and array config.
It is a perfectly valid way of describing multiple bindings at once, but it has its downsides as well:

* Large classes require developer to split binding into multiple lines,
  which is cumbersome, consumes more space than needed and doesn't have an established best practice:
  
  ```html
  <!-- each project could significantly differ on how to handle such cases -->
  <input 
    class="text-input"
    :class="{
      'text-input__active': isTextInputActive,
      'text-input__hovered': isTextInputHovered, 
    }"
  >
  ```

* Class and style bindings delegated to computeds do solve readability problem but at the cost of decreased developer experience. In order to understand what classes are actually bound to an element you'll have to scroll to a computed declaration first. And repeat that process for each element that has computed class bindings. This progressively worsens as your components become more complex.

  ```html
  <!-- actual bound classes are not instantly accessible for developer -->
  <input :class="inputClasses">
  ```

* Switching between object and array configs is redundant.
  If you have been using one of them and now need to switch to another one that would mean rewriting the whole binging to a new config type.
  An escape hatch has been designed to solve this problem that allowed object configs within array configs.
  This makes array config harder to understand if string and object configs are mixed. Finally it is hard to reason preference of one config over another.

  ```html
  <!-- which one is string and which one is object? -->
  <input :class="[activeClass, hoveredClass]">
  ```

* Runtime solutions to generate styles and classes do circumvent some of the issues above but at the same time consume part of the performance. This also gets worse the larger your app gets.

## Unfitting Custom Properties

Object config is not the best candidate when it comes to deal with Custom Properties.
Primarily because you should always wrap them in quotes and thus can not use property shorthands.
If you need to add a custom property binding you'll always have to use an object config for that property.

```html
<!-- there is just no other way of doing this -->
<input :style="{ '--custom-property': customProperty }">
<input :style="[{ '--custom-property': customProperty }, ...someOtherStyles]">
```

## Repetitive unit assignments

Each time you need to bind a CSS Value unit you'll have to go trough a step of converting it to a CSS unit representation first. This seriously limits opportunities for property shorthands. It is a recurrent issue when dealing with style bindings since in JavaScript we primarily operate on numbers and not on CSS Units when we deal with numeric values.

```html
<!-- it's either this or a computed property -->
<input :style="{ fontSize: fontSize + 'px' }">
```

# Detailed design

To deal with the issues described above class and style bindings should accept a secondary named argument that will represent a class name or a style property accordingly. A binding value should only be a property on the template render context. All of the transformations below are purely a syntax sugar and do not require any changes for existing runtime.

## Class bindings

The following examples produce equal results:

```html
<input :class:is-active="isActive">
<input :class="{ 'is-active': isActive }">
```

```html
<!-- dynamic class evaluates to a string here -->
<input :class:[dynamicClass]>
<input :class="[dynamicClass]">
```

```html
<input :class:[dynamicClass]="isActive">
<input :class="{ [dynamicClass]: isActive }">
```

```html
<input
  class="text-input"
  :class:is-active="isActive"
  :class:[colorClass]
  :class:[hoverClass]="isHovered"
>
<input
  class="text-input"
  :class="{
    'is-active': isActive,
    [colorClass]: true,
    [hoverClass]: isHovered
  }"
>
```

## Style bindings

The same goes for the following example:

```html
<input :style:color="inputColor">
<input :style="{ color: inputColor }">
```

For named style arguments these modifiers will be introduced:

* `.prop` — converts an argument to a CSS Custom Property
* `.px` — converts value to a pixel length unit
* `.rel` — converts value to a percentage unit
* `.deg` — converts value to a degree angle unit
* `.rad` — converts value to a radian angle unit
* `.grad` — converts value to a gradian angle unit
* `.turn` — converts value to a turn angle unit
* `.s` — converts value to a time in seconds unit
* `.ms` — converts value to a time in milliseconds unit

Other units are not supported in favour of simplicity.
These modifiers will only work if style binding has a secondary argument.

Examples of new modifiers and their compiled variants:

```html
<input :style:width.px="width">
<input :style="{ width: width + 'px' }">
```

```html
<input :style:input-size.prop="inputSize">
<input :style="{ '--input-size': inputSize }">
```

### Combinations

It should be possible to use object and array bindings together with secondary argument bindings for both style and class.

```html
<input :class="{ isActive }" :class:is-valid="isValid">
```

### Edge cases

CSS Custom Properties allow for an unnamed property (`color: var(--)`). It is not supported in this RFC.

# Drawbacks

* Since these additions are purely a syntax sugar for existing API this change won't change anything for render functions.
* Increased API surface.

# Alternatives

Unknown.

# Adoption strategy

This is purely an additive feature. Although a compiler support would be required. No special changes in syntax highlighting are required since it is still a valid HTML syntax.

# Unresolved questions

* Should implicit value binding for class and\or style arguments be supported?

  ```html
  <!-- these are equal -->
  <input :style:custom-property.prop>
  <input :style:custom-property.prop="customProperty">

  <!-- these are equal -->
  <input :class:is-active>
  <input :class:is-active="isActive">
  ```

* Should it be a `.percent` modifier for percentage units instead of `.rel`?
  `.rel` was initially chosen as a more concise alternative.

* Should color units be considered as well? Should they support interconversion?