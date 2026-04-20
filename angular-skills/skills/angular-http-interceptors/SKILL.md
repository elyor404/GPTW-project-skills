---
name: angular-http-interceptors
description: HTTP interceptor patterns for Angular - JWT token attachment, error handling, tenant context, loading states. Use when configuring API communication.
---

# Angular HTTP Interceptors

## When to Use This Skill

Use this skill when:
- Attaching JWT tokens to requests
- Adding tenant context headers
- Handling HTTP errors globally
- Implementing loading indicators
- Logging API calls

## Functional Interceptors (Angular 17+)

### Auth Interceptor

```typescript
// src/app/core/interceptors/auth.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { MsalService } from '@azure/msal-angular';
import { from, switchMap } from 'rxjs';
import { environment } from '../../../environments/environment';

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const msalService = inject(MsalService);

  // Only add token for API requests
  if (!req.url.startsWith(environment.apiUrl)) {
    return next(req);
  }

  const account = msalService.instance.getActiveAccount();
  if (!account) {
    return next(req);
  }

  // Silently acquire token
  return from(
    msalService.instance.acquireTokenSilent({
      scopes: ['api://gptw-plus-api/access_as_user'],
      account,
    })
  ).pipe(
    switchMap(result => {
      const authReq = req.clone({
        setHeaders: {
          Authorization: `Bearer ${result.accessToken}`,
        },
      });
      return next(authReq);
    })
  );
};
```

### Tenant Interceptor

```typescript
// src/app/core/interceptors/tenant.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../auth/auth.service';
import { environment } from '../../../environments/environment';

export const tenantInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  // Only add tenant header for API requests
  if (!req.url.startsWith(environment.apiUrl)) {
    return next(req);
  }

  const authService = inject(AuthService);
  const companyId = authService.currentUser()?.companyId;

  if (companyId) {
    const tenantReq = req.clone({
      setHeaders: {
        'X-Tenant-Id': companyId,
      },
    });
    return next(tenantReq);
  }

  return next(req);
};
```

### Error Handling Interceptor

```typescript
// src/app/core/interceptors/error.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from '../services/notification.service';
import { AuthService } from '../auth/auth.service';

export const errorInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const router = inject(Router);
  const notificationService = inject(NotificationService);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An unexpected error occurred';

      switch (error.status) {
        case 0:
          errorMessage = 'Unable to connect to the server. Please check your internet connection.';
          break;

        case 400:
          // Handle validation errors
          if (error.error?.errors) {
            const validationErrors = Object.values(error.error.errors).flat();
            errorMessage = validationErrors.join(', ');
          } else {
            errorMessage = error.error?.detail || 'Invalid request';
          }
          break;

        case 401:
          // Unauthorized - redirect to login
          authService.logout();
          return throwError(() => error);

        case 403:
          errorMessage = 'You do not have permission to perform this action';
          router.navigate(['/forbidden']);
          break;

        case 404:
          errorMessage = 'The requested resource was not found';
          break;

        case 409:
          errorMessage = error.error?.detail || 'A conflict occurred';
          break;

        case 429:
          errorMessage = 'Too many requests. Please try again later.';
          break;

        case 500:
        case 502:
        case 503:
          errorMessage = 'A server error occurred. Please try again later.';
          break;
      }

      // Don't show notification for 401 (handled by logout)
      if (error.status !== 401) {
        notificationService.showError(errorMessage);
      }

      return throwError(() => ({
        ...error,
        userMessage: errorMessage,
      }));
    })
  );
};
```

### Loading Interceptor

```typescript
// src/app/core/interceptors/loading.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs';
import { LoadingService } from '../services/loading.service';

export const loadingInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const loadingService = inject(LoadingService);

  // Skip loading indicator for certain requests
  if (req.headers.has('X-Skip-Loading')) {
    const cleanReq = req.clone({
      headers: req.headers.delete('X-Skip-Loading'),
    });
    return next(cleanReq);
  }

  loadingService.show();

  return next(req).pipe(
    finalize(() => {
      loadingService.hide();
    })
  );
};

// Loading Service
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private readonly _isLoading = signal(false);
  private activeRequests = 0;

  readonly isLoading = this._isLoading.asReadonly();

  show(): void {
    this.activeRequests++;
    this._isLoading.set(true);
  }

  hide(): void {
    this.activeRequests--;
    if (this.activeRequests <= 0) {
      this.activeRequests = 0;
      this._isLoading.set(false);
    }
  }
}
```

### Request Logging Interceptor

```typescript
// src/app/core/interceptors/logging.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpHandlerFn, HttpResponse } from '@angular/common/http';
import { tap } from 'rxjs';
import { environment } from '../../../environments/environment';

export const loggingInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  // Only log in development
  if (environment.production) {
    return next(req);
  }

  const startTime = Date.now();

  return next(req).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const duration = Date.now() - startTime;
          console.log(`[HTTP] ${req.method} ${req.urlWithParams} - ${event.status} (${duration}ms)`);
        }
      },
      error: (error) => {
        const duration = Date.now() - startTime;
        console.error(`[HTTP ERROR] ${req.method} ${req.urlWithParams} - ${error.status} (${duration}ms)`, error);
      },
    })
  );
};
```

## Configuration

### Registering Interceptors

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { tenantInterceptor } from './core/interceptors/tenant.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';
import { loadingInterceptor } from './core/interceptors/loading.interceptor';
import { loggingInterceptor } from './core/interceptors/logging.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        // Order matters! First interceptor runs first on request, last on response
        loggingInterceptor,
        loadingInterceptor,
        authInterceptor,
        tenantInterceptor,
        errorInterceptor,
      ])
    ),
    // ... other providers
  ],
};
```

## API Service Pattern

```typescript
// src/app/core/services/api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable, firstValueFrom } from 'rxjs';
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = environment.apiUrl;

  get<T>(path: string, params?: Record<string, string>): Promise<T> {
    const httpParams = params ? new HttpParams({ fromObject: params }) : undefined;
    return firstValueFrom(
      this.http.get<T>(`${this.baseUrl}${path}`, { params: httpParams })
    );
  }

  post<T>(path: string, body: unknown): Promise<T> {
    return firstValueFrom(
      this.http.post<T>(`${this.baseUrl}${path}`, body)
    );
  }

  put<T>(path: string, body: unknown): Promise<T> {
    return firstValueFrom(
      this.http.put<T>(`${this.baseUrl}${path}`, body)
    );
  }

  delete<T>(path: string): Promise<T> {
    return firstValueFrom(
      this.http.delete<T>(`${this.baseUrl}${path}`)
    );
  }

  // Skip loading indicator for specific requests
  getWithoutLoading<T>(path: string): Promise<T> {
    const headers = new HttpHeaders().set('X-Skip-Loading', 'true');
    return firstValueFrom(
      this.http.get<T>(`${this.baseUrl}${path}`, { headers })
    );
  }
}
```

## References

- [Angular HTTP Client](https://angular.dev/guide/http)
- [HTTP Interceptors](https://angular.dev/guide/http/interceptors)
