# JSX Props Override Feature Analysis

## Overview
This document provides a detailed analysis of how the jsx props overriding feature works in react-i18next's `Trans` component.

## Feature Description
The Trans component allows props defined in translation strings to override props passed to components in the `components` prop. This enables translations to control component behavior, such as changing link URLs based on the active language.

## Architecture

### Key Files
- **`src/Trans.js`**: Wrapper component that integrates with React context
- **`src/TransWithoutContext.js`**: Core implementation containing the props merging logic

### Code Flow

```
Trans Component → TransWithoutContext Component
                        ↓
                  nodesToString() - Converts React children to translation key string
                        ↓
                  t(key, options) - Fetches translation from i18next
                        ↓
                  renderNodes() - Parses translation and renders React elements
                        ↓
                  mapAST() - Maps translation AST to React components
                        ↓
                  mergeProps() - **MERGES TRANSLATION PROPS WITH COMPONENT PROPS**
```

## The Specific Location of Props Replacement

### mergeProps Function
**Location**: `/src/TransWithoutContext.js`, lines 27-32

```javascript
const mergeProps = (source, target) => {
  const newTarget = { ...target };
  // translation props (source.props) should override component props (target.props)
  newTarget.props = { ...target.props, ...source.props };
  return newTarget;
};
```

**Purpose**: This function merges props from two sources, with the source props (from translation) taking precedence over target props (from component).

**Parameters**:
- `source`: Contains props from the translation string (e.g., `{ props: { href: "https://example.com/" } }`)
- `target`: The original React component (e.g., `<a href="value-to-be-overridden" />`)

**Return**: A new object with merged props where translation props override component props.

### Usage in renderNodes
**Location**: `/src/TransWithoutContext.js`, line 331

Within the `mapAST` function (lines 300-426), when processing a tag node from the translation AST:

```javascript
// Line 321: Extract attributes from translation
const props = { ...node.attrs };

// Lines 322-329: Optionally unescape HTML entities in prop values
if (shouldUnescape) {
  Object.keys(props).forEach((p) => {
    const val = props[p];
    if (isString(val)) {
      props[p] = unescape(val);
    }
  });
}

// Line 331: Merge translation props with component props
const child = Object.keys(props).length !== 0 ? mergeProps({ props }, tmp) : tmp;
```

**Logic**:
1. Props are extracted from the parsed translation HTML/AST node (`node.attrs`)
2. If props exist (length > 0), `mergeProps` is called
3. If no props exist, the original component is used as-is

## Complete Example Walkthrough

### Input
```jsx
<Trans
  i18nKey="myKey"
  components={{ 
    CustomLink: <a href="value-to-be-overridden">fallback</a>
  }}
/>
```

### Translation String
```json
{
  "myKey": "This is a <CustomLink href=\"https://example.com/\">link to example.com</CustomLink>."
}
```

### Processing Steps

1. **Parse Translation**
   - HTML string is parsed into AST at line 249
   - Result: AST with tag node for `CustomLink` with attribute `href="https://example.com/"`

2. **Map Components**
   - Component map is created: `{ CustomLink: <a href="value-to-be-overridden">fallback</a> }`

3. **Traverse AST**
   - `mapAST` iterates over translation AST nodes
   - Finds `CustomLink` tag node

4. **Extract Props**
   - Line 321: `props = { href: "https://example.com/" }`

5. **Merge Props**
   - Line 331: `mergeProps({ props: { href: "https://example.com/" } }, <a href="value-to-be-overridden">)` 
   - Result: `<a href="https://example.com/">`

6. **Render**
   - Line 266-293: Component is cloned with merged props
   - Children are processed recursively

### Output
```html
<div>
  This is a 
  <a href="https://example.com/">
    link to example.com
  </a>
  .
</div>
```

## Array Components Support

The feature also works with indexed components:

```jsx
<Trans
  i18nKey="myKey"
  components={[ <a href="value-to-be-overridden" /> ]}
/>
```

With translation:
```json
{
  "myKey": "This is a <0 href=\"https://example.com/\">link</0>."
}
```

Same merging logic applies - the `<0>` in the translation maps to `components[0]`.

## Key Implementation Details

### HTML Parsing
- Uses `html-parse-stringify` library (line 3)
- Parses translation into AST structure at line 249

### Attribute Handling
- Attributes can be extracted from translation tags
- Supports standard HTML attribute syntax: `key="value"`
- Props are unescaped if `shouldUnescape` option is enabled (lines 322-329)

### Component Lookup
The component to merge props into is located through multiple strategies (lines 311-318):
1. Array index: `reactNodes[parseInt(node.name, 10)]`
2. Named component map: `knownComponentsMap[node.name]`
3. Object property: `rootReactNode[0][node.name]`

### Merge Strategy
The merge uses JavaScript spread operator which means:
- **Later values override earlier values**
- `{ ...target.props, ...source.props }` → source wins
- This is intentional - translation props should override component props

## Testing

### Test Location
`/test/trans.render.spec.jsx`, lines 772-792

### Test Case
```javascript
it('should override component props with translation props (issue #1902)', () => {
  const { container } = render(
    <Trans
      i18nKey="myKey"
      components={{
        CustomLink: <a href="value-to-be-overridden">fallback</a>,
      }}
    />,
  );
  expect(container.firstChild).toMatchInlineSnapshot(`
    <div>
      This is a 
      <a
        href="https://example.com/"
      >
        link to example.com
      </a>
      .
    </div>
  `);
});
```

This test verifies that:
1. Component prop `href="value-to-be-overridden"` is passed
2. Translation has `href="https://example.com/"`
3. Final rendered output uses the translation's href value
4. Original component children ("fallback") are replaced by translation content

## Related Issues
- Issue #1902: Implementation of this feature

## Summary

The jsx props override feature is implemented through a simple but effective `mergeProps` function that:
1. Copies the target component
2. Spreads target props first, then source props
3. Ensures translation props override component props

This happens during the AST mapping phase when converting translated strings back into React elements, making it a seamless part of the translation rendering process.
