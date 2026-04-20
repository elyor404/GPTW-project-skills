---
name: angular-msal-auth
description: Microsoft Entra ID authentication with MSAL Angular - token management, protected routes, login/logout flows. Use when implementing authentication.
---

# MSAL Angular Authentication

## When to Use This Skill

Use this skill when:
- Configuring Microsoft Entra ID authentication
- Managing tokens and user sessions
- Protecting routes with authentication guards
- Implementing login/logout flows

## MSAL Configuration

### Installation

```bash
npm install @azure/msal-browser @azure/msal-angular
```

### Configuration Module

```typescript
// src/app/core/auth/auth-config.ts
import { MsalGuardConfiguration, MsalInterceptorConfiguration } from '@azure/msal-angular';
import { BrowserCacheLocation, InteractionType, PublicClientApplication } from '@azure/msal-browser';
import { environment } from '../../../environments/environment';

export const msalConfig = {
  auth: {
    clientId: environment.azure.clientId,
    authority: `https://login.microsoftonline.com/${environment.azure.tenantId}`,
    redirectUri: environment.azure.redirectUri,
    postLogoutRedirectUri: environment.azure.postLogoutRedirectUri,
  },
  cache: {
    cacheLocation: BrowserCacheLocation.SessionStorage,
    storeAuthStateInCookie: false,
  },
};

export const msalGuardConfig: MsalGuardConfiguration = {
  interactionType: InteractionType.Redirect,
  authRequest: {
    scopes: ['user.read', 'api://gptw-plus-api/access_as_user'],
  },
};

export const msalInterceptorConfig: MsalInterceptorConfiguration = {
  interactionType: InteractionType.Redirect,
  protectedResourceMap: new Map([
    [environment.apiUrl, ['api://gptw-plus-api/access_as_user']],
  ]),
};

export function MSALInstanceFactory(): PublicClientApplication {
  return new PublicClientApplication(msalConfig);
}
```

### App Configuration

```typescript
// src/app/app.config.ts
import { ApplicationConfig, importProvidersFrom } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptorsFromDi } from '@angular/common/http';
import {
  MSAL_GUARD_CONFIG,
  MSAL_INSTANCE,
  MSAL_INTERCEPTOR_CONFIG,
  MsalBroadcastService,
  MsalGuard,
  MsalInterceptor,
  MsalService,
} from '@azure/msal-angular';
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { MSALInstanceFactory, msalGuardConfig, msalInterceptorConfig } from './core/auth/auth-config';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptorsFromDi()),
    {
      provide: HTTP_INTERCEPTORS,
      useClass: MsalInterceptor,
      multi: true,
    },
    {
      provide: MSAL_INSTANCE,
      useFactory: MSALInstanceFactory,
    },
    {
      provide: MSAL_GUARD_CONFIG,
      useValue: msalGuardConfig,
    },
    {
      provide: MSAL_INTERCEPTOR_CONFIG,
      useValue: msalInterceptorConfig,
    },
    MsalService,
    MsalGuard,
    MsalBroadcastService,
  ],
};
```

## Authentication Service

```typescript
// src/app/core/auth/auth.service.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { MsalBroadcastService, MsalService } from '@azure/msal-angular';
import { AccountInfo, EventMessage, EventType, InteractionStatus } from '@azure/msal-browser';
import { filter, takeUntil } from 'rxjs/operators';
import { Subject } from 'rxjs';

export interface AppUser {
  id: string;
  email: string;
  name: string;
  roles: string[];
  companyId?: string;
  hasSignoffCapability: boolean;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly msalService = inject(MsalService);
  private readonly msalBroadcastService = inject(MsalBroadcastService);
  private readonly destroying$ = new Subject<void>();

  private readonly _isAuthenticated = signal(false);
  private readonly _currentUser = signal<AppUser | null>(null);
  private readonly _isLoading = signal(true);

  readonly isAuthenticated = this._isAuthenticated.asReadonly();
  readonly currentUser = this._currentUser.asReadonly();
  readonly isLoading = this._isLoading.asReadonly();

  readonly isGptwAdmin = computed(() => 
    this._currentUser()?.roles.includes('GPTWAdmin') ?? false);

  readonly isGptwConsultant = computed(() =>
    this._currentUser()?.roles.includes('Consultant') ?? false);

  readonly isCompanyAdmin = computed(() =>
    this._currentUser()?.roles.includes('CompanyAdmin') ?? false);

  constructor() {
    this.initializeAuth();
  }

  private initializeAuth(): void {
    this.msalBroadcastService.inProgress$
      .pipe(
        filter((status: InteractionStatus) => status === InteractionStatus.None),
        takeUntil(this.destroying$)
      )
      .subscribe(() => {
        this.checkAccount();
        this._isLoading.set(false);
      });

    this.msalBroadcastService.msalSubject$
      .pipe(
        filter((msg: EventMessage) => msg.eventType === EventType.LOGIN_SUCCESS),
        takeUntil(this.destroying$)
      )
      .subscribe((result: EventMessage) => {
        const payload = result.payload as { account: AccountInfo };
        this.msalService.instance.setActiveAccount(payload.account);
        this.checkAccount();
      });
  }

  private checkAccount(): void {
    const account = this.msalService.instance.getActiveAccount();
    
    if (account) {
      this._isAuthenticated.set(true);
      this._currentUser.set(this.mapAccountToUser(account));
    } else {
      const accounts = this.msalService.instance.getAllAccounts();
      if (accounts.length > 0) {
        this.msalService.instance.setActiveAccount(accounts[0]);
        this._isAuthenticated.set(true);
        this._currentUser.set(this.mapAccountToUser(accounts[0]));
      } else {
        this._isAuthenticated.set(false);
        this._currentUser.set(null);
      }
    }
  }

  private mapAccountToUser(account: AccountInfo): AppUser {
    const claims = account.idTokenClaims as Record<string, unknown>;
    
    return {
      id: account.localAccountId,
      email: account.username,
      name: account.name ?? account.username,
      roles: (claims['roles'] as string[]) ?? [],
      companyId: claims['company_id'] as string | undefined,
      hasSignoffCapability: claims['signoff_capability'] === 'true',
    };
  }

  login(): void {
    this.msalService.loginRedirect();
  }

  logout(): void {
    this.msalService.logoutRedirect();
  }

  hasRole(role: string): boolean {
    return this._currentUser()?.roles.includes(role) ?? false;
  }

  hasAnyRole(roles: string[]): boolean {
    const userRoles = this._currentUser()?.roles ?? [];
    return roles.some(role => userRoles.includes(role));
  }

  hasModuleAccess(module: string): boolean {
    const user = this._currentUser();
    if (!user) return false;

    // GPTW roles have access to all modules
    if (this.isGptwAdmin() || this.isGptwConsultant()) return true;

    // Check module access claim
    const claims = this.msalService.instance.getActiveAccount()?.idTokenClaims as Record<string, unknown>;
    const moduleAccess = (claims?.['module_access'] as string)?.split(',') ?? [];
    return moduleAccess.includes(module);
  }

  ngOnDestroy(): void {
    this.destroying$.next();
    this.destroying$.complete();
  }
}
```

## Auth Guard

```typescript
// src/app/core/auth/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  authService.login();
  return false;
};

// Role-based guard
export const roleGuard = (allowedRoles: string[]): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isAuthenticated()) {
      authService.login();
      return false;
    }

    if (authService.hasAnyRole(allowedRoles)) {
      return true;
    }

    router.navigate(['/forbidden']);
    return false;
  };
};

// Module access guard
export const moduleAccessGuard = (moduleName: string): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isAuthenticated()) {
      authService.login();
      return false;
    }

    if (authService.hasModuleAccess(moduleName)) {
      return true;
    }

    router.navigate(['/forbidden']);
    return false;
  };
};
```

## Route Configuration

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { MsalGuard } from '@azure/msal-angular';
import { authGuard, roleGuard, moduleAccessGuard } from './core/auth/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full',
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./features/dashboard/dashboard.component'),
    canActivate: [MsalGuard],
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes'),
    canActivate: [MsalGuard, roleGuard(['GPTWAdmin'])],
  },
  {
    path: 'companies/:companyId/activate',
    loadChildren: () => import('./features/activate/activate.routes'),
    canActivate: [MsalGuard, moduleAccessGuard('activate')],
  },
  {
    path: 'companies/:companyId/elevate',
    loadChildren: () => import('./features/elevate/elevate.routes'),
    canActivate: [MsalGuard, moduleAccessGuard('elevate')],
  },
  {
    path: 'forbidden',
    loadComponent: () => import('./features/errors/forbidden.component'),
  },
];
```

## Login Component

```typescript
// src/app/features/auth/login/login.component.ts
import { Component, inject } from '@angular/core';
import { AuthService } from '../../../core/auth/auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  template: `
    <div class="login-container">
      <div class="login-card">
        <img src="assets/gptw-logo.svg" alt="GPTW PLUS+" class="logo" />
        <h1>Welcome to PLUS+</h1>
        <p>Sign in with your organisation account to continue.</p>
        <button (click)="login()" class="login-button">
          Sign in with Microsoft
        </button>
      </div>
    </div>
  `,
})
export class LoginComponent {
  private readonly authService = inject(AuthService);

  login(): void {
    this.authService.login();
  }
}
```

## Environment Configuration

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'https://localhost:7001/api',
  azure: {
    clientId: 'your-client-id',
    tenantId: 'your-tenant-id',
    redirectUri: 'http://localhost:4200',
    postLogoutRedirectUri: 'http://localhost:4200',
  },
};
```

## References

- [MSAL Angular Documentation](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-angular)
- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/entra/identity-platform/)
