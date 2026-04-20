---
name: angular-routing-guards
description: Angular route guard patterns - authentication, role-based access, module access, can-deactivate. Use when protecting routes and managing navigation.
---

# Angular Routing Guards

## When to Use This Skill

Use this skill when:
- Protecting routes with authentication
- Implementing role-based access control
- Restricting module access based on packages
- Preventing navigation with unsaved changes

## Functional Guards (Angular 17+)

### Authentication Guard

```typescript
// src/app/core/guards/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router, UrlTree } from '@angular/router';
import { AuthService } from '../auth/auth.service';

export const authGuard: CanActivateFn = (): boolean | UrlTree => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // Redirect to login
  return router.createUrlTree(['/login']);
};
```

### Role Guard

```typescript
// src/app/core/guards/role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, ActivatedRouteSnapshot, Router, UrlTree } from '@angular/router';
import { AuthService } from '../auth/auth.service';

export const roleGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot
): boolean | UrlTree => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const requiredRoles = route.data['roles'] as string[];

  if (!requiredRoles || requiredRoles.length === 0) {
    return true;
  }

  if (authService.hasAnyRole(requiredRoles)) {
    return true;
  }

  return router.createUrlTree(['/forbidden']);
};

// Factory function for specific roles
export const requireRoles = (...roles: string[]): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (authService.hasAnyRole(roles)) {
      return true;
    }

    return router.createUrlTree(['/forbidden']);
  };
};
```

### Module Access Guard

```typescript
// src/app/core/guards/module-access.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, ActivatedRouteSnapshot, Router, UrlTree } from '@angular/router';
import { AuthService } from '../auth/auth.service';

export const moduleAccessGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot
): boolean | UrlTree => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const requiredModule = route.data['module'] as string;

  if (!requiredModule) {
    return true;
  }

  // GPTW roles have access to all modules
  if (authService.isGptwAdmin() || authService.isGptwConsultant()) {
    return true;
  }

  if (authService.hasModuleAccess(requiredModule)) {
    return true;
  }

  return router.createUrlTree(['/dashboard'], {
    queryParams: { error: 'module_access_denied' },
  });
};

// Factory function for specific module
export const requireModule = (moduleName: string): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (authService.hasModuleAccess(moduleName)) {
      return true;
    }

    return router.createUrlTree(['/forbidden']);
  };
};
```

### Tenant Guard

```typescript
// src/app/core/guards/tenant.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, ActivatedRouteSnapshot, Router, UrlTree } from '@angular/router';
import { AuthService } from '../auth/auth.service';

export const tenantGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot
): boolean | UrlTree => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const routeCompanyId = route.params['companyId'];

  // GPTW Admin can access any company
  if (authService.isGptwAdmin()) {
    return true;
  }

  // GPTW Consultant can access assigned companies
  // (would need to check against assigned companies list)
  if (authService.isGptwConsultant()) {
    return true; // Simplified - would need API check
  }

  // Company users can only access their own company
  const userCompanyId = authService.currentUser()?.companyId;

  if (userCompanyId === routeCompanyId) {
    return true;
  }

  return router.createUrlTree(['/forbidden']);
};
```

### Unsaved Changes Guard

```typescript
// src/app/core/guards/unsaved-changes.guard.ts
import { inject } from '@angular/core';
import { CanDeactivateFn } from '@angular/router';
import { DialogService } from '../services/dialog.service';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = async (
  component
): Promise<boolean> => {
  if (!component.hasUnsavedChanges()) {
    return true;
  }

  const dialogService = inject(DialogService);

  return dialogService.confirm({
    title: 'Unsaved Changes',
    message: 'You have unsaved changes. Are you sure you want to leave?',
    confirmText: 'Leave',
    cancelText: 'Stay',
    isDestructive: true,
  });
};
```

## Route Configuration

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';
import { roleGuard, requireRoles } from './core/guards/role.guard';
import { moduleAccessGuard, requireModule } from './core/guards/module-access.guard';
import { tenantGuard } from './core/guards/tenant.guard';
import { unsavedChangesGuard } from './core/guards/unsaved-changes.guard';

export const routes: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./features/auth/login/login.component'),
  },
  {
    path: '',
    canActivate: [authGuard],
    children: [
      {
        path: 'dashboard',
        loadComponent: () => import('./features/dashboard/dashboard.component'),
      },
      // Admin routes - GPTW Admin only
      {
        path: 'admin',
        canActivate: [requireRoles('GPTWAdmin')],
        children: [
          {
            path: 'companies',
            loadComponent: () => import('./features/admin/company-list/company-list.component'),
          },
          {
            path: 'companies/new',
            loadComponent: () => import('./features/admin/company-form/company-form.component'),
            canDeactivate: [unsavedChangesGuard],
          },
          {
            path: 'staff',
            loadComponent: () => import('./features/admin/staff-list/staff-list.component'),
          },
        ],
      },
      // Company routes with tenant guard
      {
        path: 'companies/:companyId',
        canActivate: [tenantGuard],
        children: [
          {
            path: 'team',
            canActivate: [requireRoles('GPTWAdmin', 'CompanyAdmin')],
            loadComponent: () => import('./features/team-management/team-list.component'),
          },
          {
            path: 'diagnostic',
            canActivate: [requireRoles('GPTWAdmin', 'CompanyAdmin')],
            loadComponent: () => import('./features/diagnostic-centre/diagnostic-centre.component'),
            canDeactivate: [unsavedChangesGuard],
          },
          // Module routes
          {
            path: 'activate',
            canActivate: [requireModule('activate')],
            loadChildren: () => import('./features/activate/activate.routes'),
          },
          {
            path: 'elevate',
            canActivate: [requireModule('elevate')],
            loadChildren: () => import('./features/elevate/elevate.routes'),
          },
          {
            path: 'empower',
            canActivate: [requireModule('empower')],
            loadChildren: () => import('./features/empower/empower.routes'),
          },
        ],
      },
      // Error pages
      {
        path: 'forbidden',
        loadComponent: () => import('./features/errors/forbidden.component'),
      },
    ],
  },
  {
    path: '**',
    redirectTo: 'dashboard',
  },
];
```

## Feature Routes Example

```typescript
// src/app/features/activate/activate.routes.ts
import { Routes } from '@angular/router';
import { unsavedChangesGuard } from '../../core/guards/unsaved-changes.guard';

export default [
  {
    path: '',
    redirectTo: 'posts',
    pathMatch: 'full',
  },
  {
    path: 'posts',
    loadComponent: () => import('./social-posts-dashboard/social-posts-dashboard.component'),
  },
  {
    path: 'posts/new',
    loadComponent: () => import('./social-post-editor/social-post-editor.component'),
    canDeactivate: [unsavedChangesGuard],
  },
  {
    path: 'posts/:postId',
    loadComponent: () => import('./social-post-view/social-post-view.component'),
  },
  {
    path: 'posts/:postId/edit',
    loadComponent: () => import('./social-post-editor/social-post-editor.component'),
    canDeactivate: [unsavedChangesGuard],
  },
  {
    path: 'brand-centre',
    loadComponent: () => import('./brand-centre/brand-centre.component'),
    canDeactivate: [unsavedChangesGuard],
  },
  {
    path: 'recommendations',
    loadComponent: () => import('./activate-plus/activate-plus.component'),
  },
] as Routes;
```

## References

- [Angular Route Guards](https://angular.dev/guide/routing/common-router-tasks#preventing-unauthorized-access)
- [Functional Guards](https://angular.dev/guide/routing/route-guards)
