---
tags:
  - frontend
  - react
  - forms
  - performance
last_reviewed: 2026-10-02
related_notes:
  - "[[zod-schema-validation]]"
  - "[[validation-defense-in-depth]]"
---

# React Hook Form: Uncontrolled Inputs & Dynamic Schema Validation

> **Active Recall Self-Test:**
> 1. Why does traditional controlled React state (`value={state}` + `onChange={setState}`) degrade rendering performance on complex forms with 20+ fields?
> 2. How does React Hook Form achieve isolated, zero-re-render typing using uncontrolled DOM inputs and `ref` registration?
> 3. What is the role of `<Controller />`, and why is it necessary for custom UI components (Shadcn UI, Radix Select, DatePickers)?
> 4. How does `useFieldArray` manage dynamic arrays of inputs (e.g. adding/removing product variants) without losing user input on re-render?

---

## 1. The Core Problem
### The Controlled Component Performance Death Spiral

In standard React tutorials, form state is managed via `useState`:
```jsx
const [title, setTitle] = useState('');
const [price, setPrice] = useState(0);
// ... 20 more state variables

return <input value={title} onChange={(e) => setTitle(e.target.value)} />;
```

#### Why Controlled Forms Break at Scale
- **Root-to-Leaf Re-Renders on Every Keystroke:** Every single character typed into the `title` field updates React state, triggering a re-render of the parent form component and **every single child input and dropdown below it**.
- **Input Latency & Lag:** On complex forms with 30 inputs, dynamic pricing calculators, or rich-text editors, typing feels noticeably sluggish and drops frames below 60fps.
- **Boilerplate Explosion:** Writing `value`, `onChange`, and error mapping for dozens of fields produces massive, unmaintainable components.

#### The Architectural Solution: Uncontrolled Components via RHF
React Hook Form leverages native browser DOM `ref` pointers. The DOM itself holds the current input values. RHF listens to DOM mutation events without forcing React component re-renders until submission or schema-triggered validation occurs.

---

## 2. The Mental Model

### Controlled Inputs vs. Uncontrolled Ref Architecture
```
CONTROLLED (Standard React useState):
User types "A" ──► onChange event ──► setState("A") ──► Entire Form Re-renders
                                                     ──► Virtual DOM Diffing
                                                     ──► Re-evaluates all 25 child inputs ❌

──────────────────────────────────────────────────────────────────

UNCONTROLLED (React Hook Form):
User types "A" ──► Direct DOM Mutation (<input ref={...} />)
                   No React state change!
                   Zero Component Re-renders!
                   React Hook Form collects values via refs only on submit ✅
```

### Component Integration Strategy
```
Native HTML Elements (<input>, <textarea>, <select>)
       │
       ▼
Use register("fieldName")
(Injects ref, name, onBlur, onChange directly)

──────────────────────────────────────────────────────────────────

Controlled Custom UI (Shadcn/UI Select, Slider, React-Select, DatePicker)
       │
       ▼
Use <Controller name="fieldName" control={control} render={({ field }) => ...} />
(Bridges RHF internal state with non-native custom components)
```

---

## 3. Production Code Breakdown

### A. Dynamic Product Form with Zod & `useFieldArray` (`src/components/ProductForm.jsx`)
```jsx
import React from 'react';
import { useForm, useFieldArray, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

// 1. Strict Validation Schema
const productFormSchema = z.object({
  title: z.string().min(3, 'Title must be at least 3 characters'),
  price: z.coerce.number().positive('Price must be greater than zero'),
  category: z.enum(['Clothing', 'Footwear', 'Electronics'], {
    errorMap: () => ({ message: 'Please select a valid category' }),
  }),
  // Dynamic array of variants
  variants: z
    .array(
      z.object({
        sku: z.string().min(2, 'SKU required'),
        stock: z.coerce.number().int().nonnegative('Stock cannot be negative'),
      })
    )
    .min(1, 'At least one product variant is required'),
});

export default function ProductForm({ onSubmitProduct }) {
  // 2. Initialize useForm Hook
  const {
    register,
    control,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting, isValid },
    reset,
  } = useForm({
    resolver: zodResolver(productFormSchema),
    mode: 'onTouched', // Validates when input loses focus (Optimal UX)
    defaultValues: {
      title: '',
      price: '',
      category: 'Clothing',
      variants: [{ sku: '', stock: 0 }],
    },
  });

  // 3. Dynamic Field Array for Variants
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'variants',
  });

  // Watch a field selectively without re-rendering the whole form
  const selectedCategory = watch('category');

  const onValidSubmit = async (data) => {
    await onSubmitProduct(data);
    reset(); // Reset form to default values cleanly
  };

  return (
    <form onSubmit={handleSubmit(onValidSubmit)} className="space-y-6 max-w-xl mx-auto">
      {/* 1. Standard Native Input using register */}
      <div>
        <label className="block text-sm font-medium">Product Title</label>
        <input
          {...register('title')}
          type="text"
          className="w-full border p-2 rounded"
          placeholder="e.g. Ergonomic Keyboard"
        />
        {errors.title && <p className="text-red-500 text-xs mt-1">{errors.title.message}</p>}
      </div>

      {/* 2. Coerced Numeric Input */}
      <div>
        <label className="block text-sm font-medium">Price ($)</label>
        <input
          {...register('price')}
          type="number"
          step="0.01"
          className="w-full border p-2 rounded"
        />
        {errors.price && <p className="text-red-500 text-xs mt-1">{errors.price.message}</p>}
      </div>

      {/* 3. Dynamic Variants List (useFieldArray) */}
      <div className="space-y-4">
        <h4 className="font-semibold">Variants (Category: {selectedCategory})</h4>
        {fields.map((field, index) => (
          <div key={field.id} className="flex gap-4 items-center">
            {/* ⚠️ Always use field.id as the key, NEVER array index! */}
            <input
              {...register(`variants.${index}.sku`)}
              placeholder="SKU-001"
              className="border p-2 rounded flex-1"
            />
            <input
              {...register(`variants.${index}.stock`)}
              type="number"
              placeholder="Stock"
              className="border p-2 rounded w-24"
            />
            {fields.length > 1 && (
              <button
                type="button"
                onClick={() => remove(index)}
                className="text-red-500 hover:text-red-700"
              >
                Delete
              </button>
            )}
          </div>
        ))}

        <button
          type="button"
          onClick={() => append({ sku: '', stock: 0 })}
          className="text-blue-600 text-sm font-medium"
        >
          + Add Variant
        </button>
        {errors.variants?.message && (
          <p className="text-red-500 text-xs">{errors.variants.message}</p>
        )}
      </div>

      {/* Submit Button */}
      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full bg-blue-600 text-white py-2 rounded disabled:opacity-50"
      >
        {isSubmitting ? 'Saving Product...' : 'Create Product'}
      </button>
    </form>
  );
}
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The `field.id` vs Array `index` Key Bug in `useFieldArray`
- **The Trap:** Mapping over `fields.map((field, index) => <div key={index}>)`.
- **The Bug:** If you have 3 variants and delete the middle one, React uses the array index for its reconciliation algorithm. React fails to unmount the deleted input, causing user-typed values from other inputs to visually shift or overwrite remaining items.
- **The Fix:** **ALWAYS use `key={field.id}`**, which is a stable, unique UUID generated internally by React Hook Form.

### ⚠️ Gotcha 2: The Numeric Input String Trap (`z.coerce.number()`)
- **The Trap:** Native HTML `<input type="number">` returns a string (`"49.99"`) in its `e.target.value`. Passing this to a Zod schema defined as `z.number()` results in instant validation failure (`Expected number, received string`).
- **The Fix:** Use `z.coerce.number()` in your Zod schema to automatically cast incoming string inputs into JavaScript numbers before schema validation.

### ⚠️ Gotcha 3: The Overuse of `watch()` Causing Global Re-Renders
- **The Trap:** Calling `const formValues = watch();` at the top level of a large form re-renders the entire component on every single keystroke across any field, completely destroying the performance advantage of React Hook Form.
- **The Fix:**
  - Watch specific target fields: `const category = watch('category');`
  - Or use the `<useWatch />` custom hook inside isolated child components to restrict re-renders strictly to the branch that depends on that value.
