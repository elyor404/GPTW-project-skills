---
name: angular-state-management
description: State management patterns for Angular - services with signals, BehaviorSubject patterns, NgRx for complex state. Use when managing application state.
---

# Angular State Management

## When to Use This Skill

Use this skill when:
- Managing shared state across components
- Implementing caching and data persistence
- Building complex feature state (Activate module)
- Handling loading and error states

## Signal-Based Services (Recommended for Angular 17+)

### Social Posts State Service

```typescript
// src/app/features/activate/state/social-posts.state.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { SocialPost, SocialPostState, SocialPostFilter } from '../models/social-post.model';
import { SocialPostsApiService } from '../services/social-posts-api.service';

interface SocialPostsStateModel {
  posts: SocialPost[];
  selectedPost: SocialPost | null;
  loading: boolean;
  error: string | null;
  filter: SocialPostFilter;
}

@Injectable({ providedIn: 'root' })
export class SocialPostsState {
  private readonly api = inject(SocialPostsApiService);

  // Private writable signals
  private readonly _posts = signal<SocialPost[]>([]);
  private readonly _selectedPost = signal<SocialPost | null>(null);
  private readonly _loading = signal(false);
  private readonly _error = signal<string | null>(null);
  private readonly _filter = signal<SocialPostFilter>({});

  // Public readonly signals
  readonly posts = this._posts.asReadonly();
  readonly selectedPost = this._selectedPost.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();
  readonly filter = this._filter.asReadonly();

  // Computed signals
  readonly draftPosts = computed(() =>
    this._posts().filter(p => p.state === SocialPostState.Draft)
  );

  readonly awaitingApprovalPosts = computed(() =>
    this._posts().filter(p => p.state === SocialPostState.AwaitingApproval)
  );

  readonly approvedPosts = computed(() =>
    this._posts().filter(p => p.state === SocialPostState.Approved)
  );

  readonly publishedPosts = computed(() =>
    this._posts().filter(p => p.state === SocialPostState.Published)
  );

  readonly postCounts = computed(() => ({
    draft: this.draftPosts().length,
    awaitingApproval: this.awaitingApprovalPosts().length,
    approved: this.approvedPosts().length,
    published: this.publishedPosts().length,
    total: this._posts().length,
  }));

  readonly filteredPosts = computed(() => {
    const filter = this._filter();
    let posts = this._posts();

    if (filter.state) {
      posts = posts.filter(p => p.state === filter.state);
    }
    if (filter.type) {
      posts = posts.filter(p => p.type === filter.type);
    }

    return posts;
  });

  // Actions
  async loadPosts(companyId: string): Promise<void> {
    this._loading.set(true);
    this._error.set(null);

    try {
      const posts = await this.api.getAll(companyId, this._filter());
      this._posts.set(posts);
    } catch (err) {
      this._error.set('Failed to load posts');
      console.error(err);
    } finally {
      this._loading.set(false);
    }
  }

  async createPost(companyId: string, request: CreateSocialPostRequest): Promise<SocialPost | null> {
    this._loading.set(true);

    try {
      const post = await this.api.create(companyId, request);
      this._posts.update(posts => [...posts, post]);
      return post;
    } catch (err) {
      this._error.set('Failed to create post');
      return null;
    } finally {
      this._loading.set(false);
    }
  }

  async updatePost(companyId: string, postId: string, request: UpdateSocialPostRequest): Promise<boolean> {
    this._loading.set(true);

    try {
      const updated = await this.api.update(companyId, postId, request);
      this._posts.update(posts =>
        posts.map(p => p.id === postId ? updated : p)
      );
      if (this._selectedPost()?.id === postId) {
        this._selectedPost.set(updated);
      }
      return true;
    } catch (err) {
      this._error.set('Failed to update post');
      return false;
    } finally {
      this._loading.set(false);
    }
  }

  async submitForApproval(companyId: string, postId: string): Promise<boolean> {
    try {
      await this.api.submitForApproval(companyId, postId);
      this._posts.update(posts =>
        posts.map(p => p.id === postId
          ? { ...p, state: SocialPostState.AwaitingApproval }
          : p)
      );
      return true;
    } catch {
      this._error.set('Failed to submit for approval');
      return false;
    }
  }

  async approve(companyId: string, postId: string): Promise<boolean> {
    try {
      await this.api.approve(companyId, postId);
      this._posts.update(posts =>
        posts.map(p => p.id === postId
          ? { ...p, state: SocialPostState.Approved }
          : p)
      );
      return true;
    } catch {
      this._error.set('Failed to approve post');
      return false;
    }
  }

  async publish(companyId: string, postId: string): Promise<boolean> {
    try {
      await this.api.publish(companyId, postId);
      this._posts.update(posts =>
        posts.map(p => p.id === postId
          ? { ...p, state: SocialPostState.Published }
          : p)
      );
      return true;
    } catch {
      this._error.set('Failed to publish post');
      return false;
    }
  }

  selectPost(post: SocialPost | null): void {
    this._selectedPost.set(post);
  }

  setFilter(filter: SocialPostFilter): void {
    this._filter.set(filter);
  }

  clearError(): void {
    this._error.set(null);
  }

  reset(): void {
    this._posts.set([]);
    this._selectedPost.set(null);
    this._loading.set(false);
    this._error.set(null);
    this._filter.set({});
  }
}
```

### Using State in Components

```typescript
// src/app/features/activate/social-posts-dashboard/social-posts-dashboard.component.ts
import { Component, inject, OnInit, effect } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ActivatedRoute } from '@angular/router';
import { SocialPostsState } from '../state/social-posts.state';
import { SocialPostState } from '../models/social-post.model';

@Component({
  selector: 'app-social-posts-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="dashboard">
      <!-- Stats Cards -->
      <div class="stats-grid">
        <div class="stat-card">
          <span class="stat-value">{{ state.postCounts().draft }}</span>
          <span class="stat-label">Drafts</span>
        </div>
        <div class="stat-card">
          <span class="stat-value">{{ state.postCounts().awaitingApproval }}</span>
          <span class="stat-label">Awaiting Approval</span>
        </div>
        <div class="stat-card">
          <span class="stat-value">{{ state.postCounts().approved }}</span>
          <span class="stat-label">Approved</span>
        </div>
        <div class="stat-card">
          <span class="stat-value">{{ state.postCounts().published }}</span>
          <span class="stat-label">Published</span>
        </div>
      </div>

      <!-- Filter -->
      <div class="filters">
        <select (change)="onFilterChange($event)">
          <option value="">All States</option>
          <option value="draft">Draft</option>
          <option value="awaiting_approval">Awaiting Approval</option>
          <option value="approved">Approved</option>
          <option value="published">Published</option>
        </select>
      </div>

      <!-- Loading State -->
      @if (state.loading()) {
        <div class="loading">Loading posts...</div>
      }

      <!-- Error State -->
      @if (state.error()) {
        <div class="error-banner">
          {{ state.error() }}
          <button (click)="state.clearError()">Dismiss</button>
        </div>
      }

      <!-- Posts List -->
      <div class="posts-list">
        @for (post of state.filteredPosts(); track post.id) {
          <div class="post-card" (click)="selectPost(post)">
            <div class="post-type">{{ post.type }}</div>
            <div class="post-content">{{ post.content | slice:0:100 }}...</div>
            <div class="post-state" [class]="post.state">{{ post.state }}</div>
          </div>
        } @empty {
          <div class="empty-state">No posts found</div>
        }
      </div>
    </div>
  `,
})
export class SocialPostsDashboardComponent implements OnInit {
  readonly state = inject(SocialPostsState);
  private readonly route = inject(ActivatedRoute);

  private companyId = '';

  constructor() {
    // Effect to reload when filter changes
    effect(() => {
      const filter = this.state.filter();
      if (this.companyId) {
        this.state.loadPosts(this.companyId);
      }
    });
  }

  ngOnInit(): void {
    this.companyId = this.route.snapshot.params['companyId'];
    this.state.loadPosts(this.companyId);
  }

  onFilterChange(event: Event): void {
    const value = (event.target as HTMLSelectElement).value;
    this.state.setFilter({
      state: value ? value as SocialPostState : undefined,
    });
  }

  selectPost(post: SocialPost): void {
    this.state.selectPost(post);
  }
}
```

## BehaviorSubject Pattern (Alternative)

```typescript
// For teams preferring RxJS patterns
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SocialPostsStore {
  private readonly postsSubject = new BehaviorSubject<SocialPost[]>([]);
  private readonly loadingSubject = new BehaviorSubject<boolean>(false);
  private readonly filterSubject = new BehaviorSubject<SocialPostFilter>({});

  readonly posts$ = this.postsSubject.asObservable();
  readonly loading$ = this.loadingSubject.asObservable();
  readonly filter$ = this.filterSubject.asObservable();

  readonly filteredPosts$ = combineLatest([this.posts$, this.filter$]).pipe(
    map(([posts, filter]) => {
      if (!filter.state) return posts;
      return posts.filter(p => p.state === filter.state);
    })
  );

  readonly postCounts$ = this.posts$.pipe(
    map(posts => ({
      draft: posts.filter(p => p.state === SocialPostState.Draft).length,
      awaitingApproval: posts.filter(p => p.state === SocialPostState.AwaitingApproval).length,
      approved: posts.filter(p => p.state === SocialPostState.Approved).length,
      published: posts.filter(p => p.state === SocialPostState.Published).length,
    }))
  );

  setPosts(posts: SocialPost[]): void {
    this.postsSubject.next(posts);
  }

  setLoading(loading: boolean): void {
    this.loadingSubject.next(loading);
  }

  setFilter(filter: SocialPostFilter): void {
    this.filterSubject.next(filter);
  }

  updatePost(updated: SocialPost): void {
    const posts = this.postsSubject.value.map(p =>
      p.id === updated.id ? updated : p
    );
    this.postsSubject.next(posts);
  }
}
```

## Loading and Error State Pattern

```typescript
// src/app/shared/state/async-state.ts
export interface AsyncState<T> {
  data: T;
  loading: boolean;
  error: string | null;
}

export function createAsyncState<T>(initialData: T): AsyncState<T> {
  return {
    data: initialData,
    loading: false,
    error: null,
  };
}

// Usage with signals
@Injectable({ providedIn: 'root' })
export class DataState<T> {
  private readonly _state = signal<AsyncState<T>>(createAsyncState(null as T));

  readonly data = computed(() => this._state().data);
  readonly loading = computed(() => this._state().loading);
  readonly error = computed(() => this._state().error);

  setLoading(): void {
    this._state.update(s => ({ ...s, loading: true, error: null }));
  }

  setData(data: T): void {
    this._state.set({ data, loading: false, error: null });
  }

  setError(error: string): void {
    this._state.update(s => ({ ...s, loading: false, error }));
  }
}
```

## References

- [Angular Signals](https://angular.dev/guide/signals)
- [Angular Service Patterns](https://angular.dev/guide/architecture-services)
