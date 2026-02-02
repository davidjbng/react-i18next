# JSX Props Override Feature - Documentation Index

This directory contains comprehensive documentation about how the JSX props override feature works in react-i18next's `Trans` component.

## 📋 Quick Answer

**Q: Where are jsx props replaced in the Trans component?**

**A**: In `/src/TransWithoutContext.js` at lines 27-32 in the `mergeProps` function:

```javascript
const mergeProps = (source, target) => {
  const newTarget = { ...target };
  // translation props (source.props) should override component props (target.props)
  newTarget.props = { ...target.props, ...source.props };
  return newTarget;
};
```

This function is called during the AST processing at line 331 of the same file.

---

## 📚 Documentation Files

### 1. [JSX_PROPS_OVERRIDE_SUMMARY.md](./JSX_PROPS_OVERRIDE_SUMMARY.md)
**Start here!** Quick overview with examples.

**Contains:**
- Quick answer to "where are props replaced?"
- Simple explanation with code examples
- Common use cases
- Test information

**Best for:** Getting a quick understanding of the feature.

---

### 2. [JSX_PROPS_OVERRIDE_FLOWCHART.md](./JSX_PROPS_OVERRIDE_FLOWCHART.md)
**Visual learners!** Step-by-step diagrams.

**Contains:**
- Complete ASCII flowchart of the process
- Visual representation of data flow
- Multiple examples with diagrams
- Component lookup strategy diagram

**Best for:** Understanding the complete flow visually.

---

### 3. [JSX_PROPS_OVERRIDE_ANALYSIS.md](./JSX_PROPS_OVERRIDE_ANALYSIS.md)
**Deep dive!** Complete technical analysis.

**Contains:**
- Detailed architecture explanation
- Complete code flow with line numbers
- Implementation details
- Related issues and version info

**Best for:** Understanding the implementation in depth or maintaining the code.

---

## 🎯 Feature Overview

The jsx props override feature allows translation strings to override component props in the `Trans` component:

### Example

```jsx
// Component with default props
<Trans
  i18nKey="myKey"
  components={{ 
    CustomLink: <a href="default-url">fallback text</a>
  }}
/>
```

```json
// Translation with overriding props
{
  "myKey": "Visit <CustomLink href=\"https://example.com/\">our site</CustomLink>"
}
```

```jsx
// Result: props from translation override component props
<div>
  Visit <a href="https://example.com/">our site</a>
</div>
```

The `href` changed from `"default-url"` to `"https://example.com/"` because the translation provided a new value.

---

## 🔍 How to Use This Documentation

**If you want to:**

- **Understand the feature quickly** → Read `JSX_PROPS_OVERRIDE_SUMMARY.md`
- **See visual flow diagrams** → Read `JSX_PROPS_OVERRIDE_FLOWCHART.md`
- **Deep dive into implementation** → Read `JSX_PROPS_OVERRIDE_ANALYSIS.md`
- **Find specific code location** → Any file has the answer (lines 27-32 in `src/TransWithoutContext.js`)
- **See test cases** → All files reference the test at `test/trans.render.spec.jsx:772-792`

---

## 📂 Key Source Files

| File | Purpose |
|------|---------|
| `src/Trans.js` | Wrapper component that adds React context support |
| `src/TransWithoutContext.js` | Core implementation with `mergeProps` function |
| `test/trans.render.spec.jsx` | Test suite including props override test |
| `test/i18n.js` | Test translations including examples with props |

---

## 🔑 Key Concepts

### 1. Props Merging Strategy
Translation props override component props using JavaScript spread syntax:
```javascript
{ ...componentProps, ...translationProps }
```
Later values override earlier values.

### 2. AST Processing
- Translation strings are parsed into HTML Abstract Syntax Tree (AST)
- Props are extracted from AST nodes
- Props are merged during AST-to-React mapping

### 3. Component Lookup
Components can be specified as:
- **Array**: `components={[<a />]}` → Referenced as `<0>`
- **Object**: `components={{ Link: <a /> }}` → Referenced as `<Link>`

### 4. Attribute Support
Any valid HTML attribute can be overridden:
- `href`, `target`, `rel` for links
- `type`, `disabled` for buttons
- `aria-*` for accessibility
- Custom props for custom components

---

## 🧪 Testing

The feature is tested in the main test suite:

**File**: `test/trans.render.spec.jsx`  
**Test**: "should override component props with translation props (issue #1902)"  
**Lines**: 772-792

Run tests:
```bash
npm test trans.render.spec.jsx
```

---

## 📊 Version Information

- **Feature introduced**: v11.5.0
- **Related issue**: #1902
- **Status**: ✅ Fully implemented and tested

---

## 🤝 Contributing

When modifying the props override feature:

1. Update the `mergeProps` function in `src/TransWithoutContext.js`
2. Ensure backward compatibility
3. Update or add tests in `test/trans.render.spec.jsx`
4. Update this documentation if behavior changes

---

## 📞 Support

For questions or issues:
- Check the [main README](./README.md)
- Review existing tests for examples
- Open an issue on GitHub

---

## Summary

The jsx props override feature is a powerful tool for internationalizing React applications. It allows translators to control component behavior through translation strings, enabling features like:

- Localized URLs for different languages
- Language-specific component behavior
- Accessibility attributes in translations
- Dynamic prop values based on locale

The implementation is clean, well-tested, and integrated seamlessly into the Trans component's rendering pipeline through the `mergeProps` function.
