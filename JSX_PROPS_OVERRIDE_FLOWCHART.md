# JSX Props Override - Visual Flow Diagram

## High-Level Process Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. TRANS COMPONENT INPUT                                        │
├─────────────────────────────────────────────────────────────────┤
│ <Trans                                                          │
│   i18nKey="myKey"                                               │
│   components={{                                                 │
│     CustomLink: <a href="value-to-be-overridden">fallback</a>  │
│   }}                                                            │
│ />                                                              │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. FETCH TRANSLATION STRING                                     │
├─────────────────────────────────────────────────────────────────┤
│ t("myKey") returns:                                             │
│ "This is a <CustomLink href=\"https://example.com/\">          │
│  link to example.com</CustomLink>."                             │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. PARSE TRANSLATION TO HTML AST                                │
│    (TransWithoutContext.js, line 249)                           │
├─────────────────────────────────────────────────────────────────┤
│ HTML.parse() creates Abstract Syntax Tree:                      │
│                                                                 │
│ [                                                               │
│   { type: 'text', content: 'This is a ' },                      │
│   {                                                             │
│     type: 'tag',                                                │
│     name: 'CustomLink',                                         │
│     attrs: { href: 'https://example.com/' },  ◄── Props here!  │
│     children: [                                                 │
│       { type: 'text', content: 'link to example.com' }          │
│     ]                                                           │
│   },                                                            │
│   { type: 'text', content: '.' }                                │
│ ]                                                               │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. MAP AST TO REACT COMPONENTS                                  │
│    (TransWithoutContext.js, mapAST function, line 300)          │
├─────────────────────────────────────────────────────────────────┤
│ For each AST node:                                              │
│   if node.type === 'tag':                                       │
│     - Find matching component from components prop              │
│     - Extract props from node.attrs                             │
│     - Merge props with component                                │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. EXTRACT PROPS FROM AST NODE                                  │
│    (TransWithoutContext.js, line 321)                           │
├─────────────────────────────────────────────────────────────────┤
│ const props = { ...node.attrs };                                │
│                                                                 │
│ Result:                                                         │
│ props = { href: "https://example.com/" }                        │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. MERGE PROPS (THE KEY STEP!)                                  │
│    (TransWithoutContext.js, line 331)                           │
├─────────────────────────────────────────────────────────────────┤
│ const child = Object.keys(props).length !== 0                   │
│   ? mergeProps({ props }, tmp)                                  │
│   : tmp;                                                        │
│                                                                 │
│ Where:                                                          │
│ - tmp = <a href="value-to-be-overridden">fallback</a>          │
│ - props = { href: "https://example.com/" }                      │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. mergeProps FUNCTION                                          │
│    (TransWithoutContext.js, lines 27-32)                        │
├─────────────────────────────────────────────────────────────────┤
│ const mergeProps = (source, target) => {                        │
│   const newTarget = { ...target };                              │
│   newTarget.props = {                                           │
│     ...target.props,     // href: "value-to-be-overridden"     │
│     ...source.props      // href: "https://example.com/"       │
│   };                                                            │
│   return newTarget;                                             │
│ };                                                              │
│                                                                 │
│ Result: newTarget.props = { href: "https://example.com/" }     │
│         ↑                                                       │
│         └── Translation prop OVERRIDES component prop!         │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. CLONE AND RENDER COMPONENT                                   │
│    (TransWithoutContext.js, lines 266-293)                      │
├─────────────────────────────────────────────────────────────────┤
│ cloneElement(                                                   │
│   child,                     // <a> element                     │
│   { key: i, ...props },      // merged props                    │
│   inner                      // children: "link to example.com" │
│ )                                                               │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. FINAL OUTPUT                                                 │
├─────────────────────────────────────────────────────────────────┤
│ <div>                                                           │
│   This is a                                                     │
│   <a href="https://example.com/">                               │
│     link to example.com                                         │
│   </a>                                                          │
│   .                                                             │
│ </div>                                                          │
│                                                                 │
│ ✓ href changed from "value-to-be-overridden" to               │
│   "https://example.com/"                                        │
│ ✓ Children changed from "fallback" to "link to example.com"    │
└─────────────────────────────────────────────────────────────────┘
```

## The Critical mergeProps Function

```javascript
// Location: src/TransWithoutContext.js, lines 27-32

const mergeProps = (source, target) => {
  const newTarget = { ...target };
  // Translation props (source.props) override component props (target.props)
  newTarget.props = { ...target.props, ...source.props };
  return newTarget;
};
```

### Why This Works

In JavaScript object spread syntax:
```javascript
{ ...A, ...B }
```
Properties in `B` override properties in `A` when they have the same key.

So:
```javascript
{
  ...{ href: "value-to-be-overridden" },  // Component prop (gets overridden)
  ...{ href: "https://example.com/" }     // Translation prop (wins!)
}
// Result: { href: "https://example.com/" }
```

## Props Override Examples

### Example 1: Single Prop Override
```jsx
// Component
components={{ Link: <a href="/old" className="link" /> }}

// Translation
"Go <Link href=\"/new\">here</Link>"

// Result
<a href="/new" className="link">here</a>
         ↑                ↑
    Overridden       Preserved
```

### Example 2: Multiple Props Override
```jsx
// Component
components={{ Button: <button type="button" disabled /> }}

// Translation  
"Click <Button type=\"submit\" aria-label=\"Send\">Send</Button>"

// Result
<button type="submit" disabled aria-label="Send">Send</button>
             ↑            ↑            ↑
        Overridden  Preserved    Added from translation
```

### Example 3: Array Components (Indexed)
```jsx
// Component
components={[
  <a href="/page1" />,
  <a href="/page2" />
]}

// Translation
"Visit <0 href=\"/updated1\">first</0> and <1 href=\"/updated2\">second</1>"

// Result
Visit <a href="/updated1">first</a> and <a href="/updated2">second</a>
```

## Component Lookup Strategy

When the translation references a component, it's looked up in this order:

```
1. Array Index (if components is an array)
   components[parseInt(node.name, 10)]
   Example: <0> looks up components[0]

2. Named Map (if components is an object)
   knownComponentsMap[node.name]
   Example: <CustomLink> looks up components.CustomLink

3. Nested Object Property
   rootReactNode[0][node.name]
   Fallback for special cases
```

## Key Files and Line Numbers

| File | Lines | Purpose |
|------|-------|---------|
| `src/TransWithoutContext.js` | 27-32 | `mergeProps` function definition |
| `src/TransWithoutContext.js` | 249 | Parse translation string to AST |
| `src/TransWithoutContext.js` | 300-426 | `mapAST` function - main processing |
| `src/TransWithoutContext.js` | 321 | Extract props from AST node |
| `src/TransWithoutContext.js` | 331 | Call `mergeProps` to override props |
| `src/Trans.js` | 7-45 | Wrapper that adds React context support |
| `test/trans.render.spec.jsx` | 772-792 | Test case for props override feature |
| `test/i18n.js` | 19-20 | Example translation with props |
