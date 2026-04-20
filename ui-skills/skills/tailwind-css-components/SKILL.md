---
name: tailwind-css-components
description: Tailwind CSS component patterns for rapid UI development - utility classes, responsive design, common components. Use when styling Angular components.
---

# Tailwind CSS Components

## When to Use This Skill

Use this skill when:
- Styling Angular components with Tailwind
- Building responsive layouts
- Creating reusable UI patterns
- Implementing consistent design system

## Setup with Angular

```bash
ng add tailwindcss
```

## Common Component Patterns

### Card Component

```html
<!-- Stats Card -->
<div class="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
  <div class="flex items-center justify-between">
    <div>
      <p class="text-sm font-medium text-grey-500">Total Posts</p>
      <p class="text-3xl font-bold text-grey-900">124</p>
    </div>
    <div class="p-3 bg-blue-100 rounded-full">
      <svg class="w-6 h-6 text-blue-600"><!-- icon --></svg>
    </div>
  </div>
  <p class="mt-2 text-sm text-green-600">
    <span class="font-medium">+12%</span> from last month
  </p>
</div>

<!-- Content Card -->
<div class="bg-white rounded-lg shadow border border-grey-200 overflow-hidden">
  <div class="px-6 py-4 border-b border-grey-200">
    <h3 class="text-lg font-semibold text-grey-900">Card Title</h3>
  </div>
  <div class="p-6">
    <!-- Content -->
  </div>
  <div class="px-6 py-4 bg-grey-50 border-t border-grey-200">
    <button class="text-blue-600 hover:text-blue-800 font-medium">
      View Details →
    </button>
  </div>
</div>
```

### Button Styles

```html
<!-- Primary Button -->
<button class="px-4 py-2 bg-blue-600 text-white font-medium rounded-md 
               hover:bg-blue-700 focus:outline-none focus:ring-2 
               focus:ring-blue-500 focus:ring-offset-2 
               disabled:opacity-50 disabled:cursor-not-allowed
               transition-colors">
  Save Changes
</button>

<!-- Secondary Button -->
<button class="px-4 py-2 bg-white text-grey-700 font-medium rounded-md 
               border border-grey-300 hover:bg-grey-50 
               focus:outline-none focus:ring-2 focus:ring-blue-500">
  Cancel
</button>

<!-- Danger Button -->
<button class="px-4 py-2 bg-red-600 text-white font-medium rounded-md 
               hover:bg-red-700 focus:outline-none focus:ring-2 
               focus:ring-red-500 focus:ring-offset-2">
  Delete
</button>

<!-- Icon Button -->
<button class="p-2 text-grey-500 hover:text-grey-700 hover:bg-grey-100 
               rounded-full transition-colors">
  <svg class="w-5 h-5"><!-- icon --></svg>
</button>
```

### Form Elements

```html
<!-- Text Input -->
<div class="space-y-1">
  <label class="block text-sm font-medium text-grey-700">
    Company Name
  </label>
  <input 
    type="text" 
    class="w-full px-3 py-2 border border-grey-300 rounded-md shadow-sm
           focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500
           disabled:bg-grey-100 disabled:cursor-not-allowed"
    placeholder="Enter company name"
  />
  <p class="text-sm text-red-600">Company name is required</p>
</div>

<!-- Select -->
<div class="space-y-1">
  <label class="block text-sm font-medium text-grey-700">Status</label>
  <select class="w-full px-3 py-2 border border-grey-300 rounded-md shadow-sm
                 focus:outline-none focus:ring-2 focus:ring-blue-500">
    <option>Select status</option>
    <option>Draft</option>
    <option>Published</option>
  </select>
</div>

<!-- Checkbox -->
<label class="flex items-center space-x-3">
  <input 
    type="checkbox" 
    class="w-4 h-4 text-blue-600 border-grey-300 rounded
           focus:ring-2 focus:ring-blue-500"
  />
  <span class="text-sm text-grey-700">Enable AI features</span>
</label>
```

### Status Badges

```html
<!-- Draft -->
<span class="px-2.5 py-0.5 text-xs font-medium rounded-full 
             bg-grey-100 text-grey-800">
  Draft
</span>

<!-- Awaiting Approval -->
<span class="px-2.5 py-0.5 text-xs font-medium rounded-full 
             bg-yellow-100 text-yellow-800">
  Awaiting Approval
</span>

<!-- Approved -->
<span class="px-2.5 py-0.5 text-xs font-medium rounded-full 
             bg-green-100 text-green-800">
  Approved
</span>

<!-- Published -->
<span class="px-2.5 py-0.5 text-xs font-medium rounded-full 
             bg-blue-100 text-blue-800">
  Published
</span>
```

### Navigation

```html
<!-- Sidebar Navigation -->
<nav class="w-64 bg-grey-900 min-h-screen">
  <div class="px-4 py-6">
    <img src="logo.svg" alt="GPTW PLUS+" class="h-8" />
  </div>
  <ul class="space-y-1 px-3">
    <li>
      <a href="#" class="flex items-center px-3 py-2 text-white 
                         bg-grey-800 rounded-md">
        <svg class="w-5 h-5 mr-3"><!-- icon --></svg>
        Dashboard
      </a>
    </li>
    <li>
      <a href="#" class="flex items-center px-3 py-2 text-grey-300 
                         hover:text-white hover:bg-grey-800 rounded-md">
        <svg class="w-5 h-5 mr-3"><!-- icon --></svg>
        Activate
      </a>
    </li>
  </ul>
</nav>
```

## Responsive Patterns

### Desktop-First Responsive

```html
<!-- Desktop first (default desktop, override for smaller) -->
<div class="grid grid-cols-4 md:grid-cols-2 sm:grid-cols-1 gap-6">
  <!-- Cards -->
</div>

<!-- Container widths -->
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
  <!-- Content -->
</div>
```

### Responsive Table

```html
<div class="overflow-x-auto">
  <table class="min-w-full divide-y divide-grey-200">
    <thead class="bg-grey-50">
      <tr>
        <th class="px-6 py-3 text-left text-xs font-medium text-grey-500 uppercase tracking-wider">
          Name
        </th>
        <!-- More columns -->
      </tr>
    </thead>
    <tbody class="bg-white divide-y divide-grey-200">
      <tr class="hover:bg-grey-50">
        <td class="px-6 py-4 whitespace-nowrap text-sm text-grey-900">
          Company Name
        </td>
        <!-- More cells -->
      </tr>
    </tbody>
  </table>
</div>
```

## Custom Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{html,ts}'],
  theme: {
    extend: {
      colors: {
        'gptw-blue': {
          50: '#e3f2fd',
          100: '#bbdefb',
          500: '#0066cc',
          600: '#0052a3',
          700: '#004c99',
        },
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
    },
  },
  plugins: [],
};
```

## References

- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Tailwind with Angular](https://tailwindcss.com/docs/guides/angular)
