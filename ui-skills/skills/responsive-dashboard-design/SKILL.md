---
name: responsive-dashboard-design
description: Dashboard layout patterns for admin interfaces - grid layouts, stats cards, navigation, data tables. Use when building admin dashboards.
---

# Responsive Dashboard Design

## When to Use This Skill

Use this skill when:
- Building admin dashboard layouts
- Creating stats overview sections
- Designing navigation patterns
- Implementing data tables and cards

## Dashboard Layout Structure

### Main Layout

```html
<!-- Main Dashboard Layout -->
<div class="min-h-screen bg-grey-100">
  <!-- Sidebar -->
  <aside class="fixed inset-y-0 left-0 w-64 bg-grey-900 z-50">
    <!-- Logo -->
    <div class="flex items-center h-16 px-6 bg-grey-800">
      <img src="assets/logo-white.svg" alt="GPTW PLUS+" class="h-8" />
    </div>
    
    <!-- Navigation -->
    <nav class="mt-6 px-3">
      <div class="space-y-1">
        <!-- Nav items -->
      </div>
    </nav>
  </aside>

  <!-- Main Content -->
  <div class="pl-64">
    <!-- Top Header -->
    <header class="h-16 bg-white shadow-sm flex items-center justify-between px-6">
      <h1 class="text-xl font-semibold text-grey-900">Dashboard</h1>
      
      <!-- User Menu -->
      <div class="flex items-center space-x-4">
        <span class="text-sm text-grey-600">John Smith</span>
        <div class="w-8 h-8 bg-blue-600 rounded-full flex items-center justify-center">
          <span class="text-white text-sm font-medium">JS</span>
        </div>
      </div>
    </header>

    <!-- Page Content -->
    <main class="p-6">
      <!-- Content goes here -->
    </main>
  </div>
</div>
```

### Stats Grid

```html
<!-- Stats Overview -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
  <!-- Stat Card 1 -->
  <div class="bg-white rounded-lg shadow p-6">
    <div class="flex items-center">
      <div class="flex-shrink-0 p-3 bg-blue-100 rounded-lg">
        <svg class="w-6 h-6 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
        </svg>
      </div>
      <div class="ml-4">
        <p class="text-sm font-medium text-grey-500">Total Posts</p>
        <p class="text-2xl font-bold text-grey-900">124</p>
      </div>
    </div>
    <div class="mt-4 flex items-center text-sm">
      <span class="text-green-600 font-medium">↑ 12%</span>
      <span class="text-grey-500 ml-2">from last month</span>
    </div>
  </div>

  <!-- Stat Card 2 - Awaiting Approval -->
  <div class="bg-white rounded-lg shadow p-6">
    <div class="flex items-center">
      <div class="flex-shrink-0 p-3 bg-yellow-100 rounded-lg">
        <svg class="w-6 h-6 text-yellow-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
      </div>
      <div class="ml-4">
        <p class="text-sm font-medium text-grey-500">Awaiting Approval</p>
        <p class="text-2xl font-bold text-grey-900">8</p>
      </div>
    </div>
    <div class="mt-4">
      <a href="#" class="text-sm text-blue-600 hover:text-blue-800 font-medium">
        Review now →
      </a>
    </div>
  </div>

  <!-- More stat cards... -->
</div>
```

### Recent Activity Section

```html
<!-- Recent Activity -->
<div class="bg-white rounded-lg shadow">
  <div class="px-6 py-4 border-b border-grey-200 flex items-center justify-between">
    <h2 class="text-lg font-semibold text-grey-900">Recent Activity</h2>
    <a href="#" class="text-sm text-blue-600 hover:text-blue-800">View all</a>
  </div>
  <div class="divide-y divide-grey-200">
    <!-- Activity Item -->
    <div class="px-6 py-4 hover:bg-grey-50">
      <div class="flex items-start">
        <div class="flex-shrink-0">
          <span class="inline-flex items-center justify-center w-8 h-8 rounded-full bg-green-100">
            <svg class="w-4 h-4 text-green-600" fill="currentColor" viewBox="0 0 20 20">
              <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
            </svg>
          </span>
        </div>
        <div class="ml-4 flex-1">
          <p class="text-sm text-grey-900">
            <span class="font-medium">Social post</span> was approved
          </p>
          <p class="text-sm text-grey-500">2 hours ago</p>
        </div>
      </div>
    </div>
    <!-- More items... -->
  </div>
</div>
```

### Data Table Layout

```html
<!-- Table Section -->
<div class="bg-white rounded-lg shadow overflow-hidden">
  <!-- Table Header -->
  <div class="px-6 py-4 border-b border-grey-200 flex items-center justify-between">
    <h2 class="text-lg font-semibold text-grey-900">Companies</h2>
    <div class="flex items-center space-x-4">
      <!-- Search -->
      <div class="relative">
        <input 
          type="text" 
          placeholder="Search..."
          class="pl-10 pr-4 py-2 border border-grey-300 rounded-md text-sm focus:ring-blue-500 focus:border-blue-500"
        />
        <svg class="w-5 h-5 text-grey-400 absolute left-3 top-2.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
        </svg>
      </div>
      <!-- Add Button -->
      <button class="px-4 py-2 bg-blue-600 text-white text-sm font-medium rounded-md hover:bg-blue-700">
        Add Company
      </button>
    </div>
  </div>

  <!-- Table -->
  <table class="min-w-full divide-y divide-grey-200">
    <thead class="bg-grey-50">
      <tr>
        <th class="px-6 py-3 text-left text-xs font-medium text-grey-500 uppercase tracking-wider">
          Company
        </th>
        <th class="px-6 py-3 text-left text-xs font-medium text-grey-500 uppercase tracking-wider">
          Client ID
        </th>
        <th class="px-6 py-3 text-left text-xs font-medium text-grey-500 uppercase tracking-wider">
          Status
        </th>
        <th class="px-6 py-3 text-left text-xs font-medium text-grey-500 uppercase tracking-wider">
          Packages
        </th>
        <th class="px-6 py-3 text-right text-xs font-medium text-grey-500 uppercase tracking-wider">
          Actions
        </th>
      </tr>
    </thead>
    <tbody class="bg-white divide-y divide-grey-200">
      <tr class="hover:bg-grey-50">
        <td class="px-6 py-4 whitespace-nowrap">
          <div class="flex items-center">
            <div class="flex-shrink-0 w-10 h-10 bg-grey-200 rounded-full flex items-center justify-center">
              <span class="text-grey-600 font-medium">AC</span>
            </div>
            <div class="ml-4">
              <div class="text-sm font-medium text-grey-900">Acme Corporation</div>
              <div class="text-sm text-grey-500">acme@example.com</div>
            </div>
          </div>
        </td>
        <td class="px-6 py-4 whitespace-nowrap text-sm text-grey-500">
          UK-1234
        </td>
        <td class="px-6 py-4 whitespace-nowrap">
          <span class="px-2 py-1 text-xs font-medium rounded-full bg-green-100 text-green-800">
            Active
          </span>
        </td>
        <td class="px-6 py-4 whitespace-nowrap text-sm text-grey-500">
          Activate, Elevate
        </td>
        <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
          <button class="text-blue-600 hover:text-blue-900 mr-4">Edit</button>
          <button class="text-red-600 hover:text-red-900">Delete</button>
        </td>
      </tr>
    </tbody>
  </table>

  <!-- Pagination -->
  <div class="px-6 py-4 border-t border-grey-200 flex items-center justify-between">
    <p class="text-sm text-grey-500">
      Showing <span class="font-medium">1</span> to <span class="font-medium">10</span> of <span class="font-medium">97</span> results
    </p>
    <nav class="flex items-center space-x-2">
      <button class="px-3 py-1 border border-grey-300 rounded text-sm hover:bg-grey-50">Previous</button>
      <button class="px-3 py-1 bg-blue-600 text-white rounded text-sm">1</button>
      <button class="px-3 py-1 border border-grey-300 rounded text-sm hover:bg-grey-50">2</button>
      <button class="px-3 py-1 border border-grey-300 rounded text-sm hover:bg-grey-50">3</button>
      <button class="px-3 py-1 border border-grey-300 rounded text-sm hover:bg-grey-50">Next</button>
    </nav>
  </div>
</div>
```

## Sidebar Navigation

```html
<nav class="mt-6 px-3 space-y-1">
  <!-- Dashboard -->
  <a href="#" class="flex items-center px-3 py-2 text-white bg-grey-800 rounded-md group">
    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6" />
    </svg>
    Dashboard
  </a>

  <!-- Section Header -->
  <div class="pt-4">
    <p class="px-3 text-xs font-semibold text-grey-400 uppercase tracking-wider">Modules</p>
  </div>

  <!-- Activate -->
  <a href="#" class="flex items-center px-3 py-2 text-grey-300 hover:text-white hover:bg-grey-800 rounded-md group">
    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5.882V19.24a1.76 1.76 0 01-3.417.592l-2.147-6.15M18 13a3 3 0 100-6M5.436 13.683A4.001 4.001 0 017 6h1.832c4.1 0 7.625-1.234 9.168-3v14c-1.543-1.766-5.067-3-9.168-3H7a3.988 3.988 0 01-1.564-.317z" />
    </svg>
    Activate
    <span class="ml-auto bg-grey-700 text-grey-300 px-2 py-0.5 text-xs rounded-full">8</span>
  </a>

  <!-- Elevate -->
  <a href="#" class="flex items-center px-3 py-2 text-grey-300 hover:text-white hover:bg-grey-800 rounded-md group">
    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" />
    </svg>
    Elevate
  </a>

  <!-- Empower (disabled) -->
  <a href="#" class="flex items-center px-3 py-2 text-grey-500 cursor-not-allowed rounded-md group">
    <svg class="w-5 h-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253" />
    </svg>
    Empower
    <span class="ml-auto text-xs text-grey-500">Coming Soon</span>
  </a>
</nav>
```

## References

- [Tailwind UI Components](https://tailwindui.com/)
- [Dashboard Design Patterns](https://designmodo.com/dashboard-ui-patterns/)
