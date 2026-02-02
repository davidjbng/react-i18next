# JSX Props Override Feature - Summary

## Quick Answer

**Where are JSX props replaced?**

The jsx props are replaced in the `mergeProps` function located at:

**File**: `/src/TransWithoutContext.js`  
**Lines**: 27-32

```javascript
const mergeProps = (source, target) => {
  const newTarget = { ...target };
  // translation props (source.props) should override component props (target.props)
  newTarget.props = { ...target.props, ...source.props };
  return newTarget;
};
```

This function is called during the AST mapping process at **line 331** of the same file.

---

## How It Works

### 1. Simple Explanation

When you use the `Trans` component with components that have props:

```jsx
<Trans
  i18nKey="myKey"
  components={{ CustomLink: <a href="old-value" /> }}
/>
```

And your translation has props in the tags:

```json
"myKey": "Text <CustomLink href=\"new-value\">link</CustomLink>"
```

The props from the translation (`href="new-value"`) **override** the props from the component (`href="old-value"`).

### 2. Technical Flow

1. **Input**: Component with props + Translation with props
2. **Parse**: Translation string is parsed into HTML AST
3. **Extract**: Props are extracted from the AST nodes (`node.attrs`)
4. **Merge**: `mergeProps` is called to merge props (translation props win)
5. **Render**: Component is cloned with the merged props
6. **Output**: Final React element with overridden props

### 3. Code Path

```
Trans.js (wrapper)
  ↓
TransWithoutContext.js
  ↓
nodesToString() - Convert children to string
  ↓
t(key) - Get translation
  ↓
HTML.parse() - Parse translation to AST (line 249)
  ↓
renderNodes() - Start rendering (line 209)
  ↓
mapAST() - Map AST to React (line 300)
  ↓
Extract props from AST (line 321)
  ↓
mergeProps() - OVERRIDE PROPS HERE (line 331)
  ↓
cloneElement() - Create final element (line 266)
```

---

## Key Implementation Details

### The mergeProps Function

```javascript
const mergeProps = (source, target) => {
  const newTarget = { ...target };
  // Spread target props first, then source props
  // JavaScript spread: later values override earlier values
  newTarget.props = { ...target.props, ...source.props };
  return newTarget;
};
```

**Why it works**: JavaScript object spread syntax `{ ...A, ...B }` means properties in B override properties in A.

### Usage Context

Located in the `mapAST` function when processing tag nodes:

```javascript
// Line 321: Extract attributes from translation HTML/AST
const props = { ...node.attrs };

// Lines 322-329: Optional unescaping of HTML entities
if (shouldUnescape) {
  Object.keys(props).forEach((p) => {
    const val = props[p];
    if (isString(val)) {
      props[p] = unescape(val);
    }
  });
}

// Line 331: Merge translation props with component props
const child = Object.keys(props).length !== 0 
  ? mergeProps({ props }, tmp)  // ← THE OVERRIDE HAPPENS HERE
  : tmp;
```

---

## Complete Example

### Input Code
```jsx
<Trans
  i18nKey="myKey"
  components={{ 
    CustomLink: <a href="value-to-be-overridden">fallback</a>
  }}
/>
```

### Translation File
```json
{
  "myKey": "This is a <CustomLink href=\"https://example.com/\">link to example.com</CustomLink>."
}
```

### What Happens

1. Component provides: `<a href="value-to-be-overridden">fallback</a>`
2. Translation provides: `href="https://example.com/"` and children "link to example.com"
3. `mergeProps` merges these:
   ```javascript
   {
     ...{ href: "value-to-be-overridden" },  // Component (gets overridden)
     ...{ href: "https://example.com/" }     // Translation (wins!)
   }
   ```
4. Result: `{ href: "https://example.com/" }`

### Final Output
```html
<div>
  This is a 
  <a href="https://example.com/">
    link to example.com
  </a>
  .
</div>
```

**Note**: Both the `href` prop AND the children are replaced by the translation values.

---

## When This Feature Is Useful

### Use Case 1: Localized Links
Different URLs for different languages:

```jsx
<Trans
  i18nKey="supportLink"
  components={{ Link: <a href="/support/en" /> }}
/>
```

```json
{
  "en": "Visit our <Link href=\"/support/en\">support page</Link>",
  "fr": "Visitez notre <Link href=\"/support/fr\">page de support</Link>",
  "de": "Besuchen Sie unsere <Link href=\"/support/de\">Support-Seite</Link>"
}
```

### Use Case 2: Dynamic Attributes
Control component behavior from translations:

```jsx
<Trans
  i18nKey="externalLink"
  components={{ Link: <a target="_self" /> }}
/>
```

```json
{
  "externalLink": "Check <Link href=\"https://external.com\" target=\"_blank\" rel=\"noopener\">this site</Link>"
}
```

### Use Case 3: Accessibility Attributes
Add a11y attributes in translations:

```jsx
<Trans
  i18nKey="downloadButton"
  components={{ Button: <button type="button" /> }}
/>
```

```json
{
  "downloadButton": "<Button type=\"submit\" aria-label=\"Download report\">Download</Button>"
}
```

---

## Testing

The feature is tested in `/test/trans.render.spec.jsx` at lines 772-792:

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
  
  // Verifies that href is "https://example.com/" not "value-to-be-overridden"
  expect(container.firstChild).toMatchInlineSnapshot(`
    <div>
      This is a 
      <a href="https://example.com/">
        link to example.com
      </a>
      .
    </div>
  `);
});
```

---

## Related Documentation

For more details, see:

- **JSX_PROPS_OVERRIDE_ANALYSIS.md**: Complete technical analysis
- **JSX_PROPS_OVERRIDE_FLOWCHART.md**: Visual flow diagrams and examples

---

## Version Information

This feature was introduced in **v11.5.0** as mentioned in the problem statement.

The implementation has been part of the codebase and is working as documented. The feature allows translators to control component props through translation strings, enabling more flexible internationalization of React applications.
