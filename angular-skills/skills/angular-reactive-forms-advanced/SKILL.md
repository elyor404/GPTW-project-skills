---
name: angular-reactive-forms-advanced
description: Advanced Reactive Forms patterns for complex forms - nested forms, dynamic fields, custom validators, form arrays. Use for Diagnostic Centre and Brand Centre forms.
---

# Advanced Reactive Forms Patterns

## When to Use This Skill

Use this skill when:
- Building complex multi-section forms (Diagnostic Centre)
- Creating dynamic form fields
- Implementing custom validators
- Managing form arrays (e.g., multiple brand colours)
- Handling nested form groups

## Form Structure

### Diagnostic Centre Form

```typescript
// src/app/features/diagnostic-centre/diagnostic-centre.form.ts
import { Injectable, inject } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators } from '@angular/forms';

export interface DiagnosticFormValue {
  companyInfo: {
    careersUrl: string;
    linkedInUrl: string;
    employeeHashtags: string[];
  };
  evp: {
    hasExistingEvp: boolean;
    existingEvpText: string;
    evpRecommendationsRequested: boolean;
    strategicPillars: string[];
    ergs: string[];
    talentAttractionGoals: string[];
  };
  businessPerformance: {
    profitability: PerformanceMetric;
    productivity: PerformanceMetric;
    efficiency: PerformanceMetric;
    customerSatisfaction: PerformanceMetric;
    marketResilience: PerformanceMetric;
    socialImpact: PerformanceMetric;
    talentAttraction: PerformanceMetric;
    talentRetention: PerformanceMetric;
  };
}

interface PerformanceMetric {
  score: number;
  notes: string;
}

@Injectable({ providedIn: 'root' })
export class DiagnosticFormService {
  private readonly fb = inject(FormBuilder);

  createForm(): FormGroup {
    return this.fb.group({
      companyInfo: this.fb.group({
        careersUrl: ['', [Validators.pattern(/^https?:\/\/.+/)]],
        linkedInUrl: ['', [Validators.pattern(/^https?:\/\/(www\.)?linkedin\.com\/.+/)]],
        employeeHashtags: this.fb.array([]),
      }),

      evp: this.fb.group({
        hasExistingEvp: [false],
        existingEvpText: [''],
        evpRecommendationsRequested: [false],
        strategicPillars: this.fb.array([]),
        ergs: this.fb.array([]),
        talentAttractionGoals: this.fb.array([]),
      }),

      businessContext: this.fb.group({
        majorTransformation: ['', [Validators.maxLength(2000)]],
        marketTrendsChallenges: ['', [Validators.maxLength(2000)]],
      }),

      businessPerformance: this.fb.group({
        profitability: this.createMetricGroup(),
        productivity: this.createMetricGroup(),
        efficiency: this.createMetricGroup(),
        customerSatisfaction: this.createMetricGroup(),
        marketResilience: this.createMetricGroup(),
        socialImpact: this.createMetricGroup(),
        talentAttraction: this.createMetricGroup(),
        talentRetention: this.createMetricGroup(),
      }),

      metrics: this.fb.group({
        revenue: [''],
        employeeCount: ['', [Validators.min(1)]],
        revenuePerFte: [''],
        ebitda: [''],
        netProfitMargin: [''],
        nps: ['', [Validators.min(-100), Validators.max(100)]],
        sat: ['', [Validators.min(0), Validators.max(100)]],
      }),

      cultureAudit: this.fb.group({
        keyQuality: ['', [Validators.maxLength(5000)]],
      }),
    });
  }

  private createMetricGroup(): FormGroup {
    return this.fb.group({
      score: [null, [Validators.min(1), Validators.max(5)]],
      notes: ['', [Validators.maxLength(500)]],
    });
  }
}
```

### Diagnostic Centre Component

```typescript
// src/app/features/diagnostic-centre/diagnostic-centre.component.ts
import { Component, inject, OnInit, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormGroup, FormArray } from '@angular/forms';
import { DiagnosticFormService } from './diagnostic-centre.form';
import { DiagnosticService } from './diagnostic.service';

@Component({
  selector: 'app-diagnostic-centre',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <!-- Company Info Section -->
      <section formGroupName="companyInfo">
        <h2>Company Information</h2>
        
        <div class="form-field">
          <label for="careersUrl">Careers URL</label>
          <input id="careersUrl" formControlName="careersUrl" type="url" />
          @if (form.get('companyInfo.careersUrl')?.errors?.['pattern']) {
            <span class="error">Please enter a valid URL</span>
          }
        </div>

        <div class="form-field">
          <label>Employee Hashtags</label>
          <div formArrayName="employeeHashtags">
            @for (hashtag of employeeHashtags.controls; track $index) {
              <div class="array-item">
                <input [formControlName]="$index" placeholder="#YourHashtag" />
                <button type="button" (click)="removeHashtag($index)">Remove</button>
              </div>
            }
          </div>
          <button type="button" (click)="addHashtag()">Add Hashtag</button>
        </div>
      </section>

      <!-- EVP Section -->
      <section formGroupName="evp">
        <h2>EVP & Talent Attraction</h2>
        
        <div class="form-field">
          <label>
            <input type="checkbox" formControlName="hasExistingEvp" />
            Do you have an existing EVP?
          </label>
        </div>

        @if (form.get('evp.hasExistingEvp')?.value) {
          <div class="form-field">
            <label for="existingEvpText">Describe your EVP</label>
            <textarea id="existingEvpText" formControlName="existingEvpText" rows="4"></textarea>
          </div>
        }

        <div class="form-field">
          <label>Strategic Pillars</label>
          <div formArrayName="strategicPillars">
            @for (pillar of strategicPillars.controls; track $index) {
              <div class="array-item">
                <input [formControlName]="$index" placeholder="Enter pillar" />
                <button type="button" (click)="removeStrategicPillar($index)">Remove</button>
              </div>
            }
          </div>
          <button type="button" (click)="addStrategicPillar()">Add Pillar</button>
        </div>
      </section>

      <!-- Business Performance Section -->
      <section formGroupName="businessPerformance">
        <h2>Business Performance (1-5 Scale)</h2>
        
        @for (metric of performanceMetrics; track metric.key) {
          <div [formGroupName]="metric.key" class="metric-group">
            <label>{{ metric.label }}</label>
            <div class="metric-inputs">
              <select formControlName="score">
                <option [ngValue]="null">Select score</option>
                @for (score of [1, 2, 3, 4, 5]; track score) {
                  <option [ngValue]="score">{{ score }}</option>
                }
              </select>
              <input formControlName="notes" placeholder="Additional notes" />
            </div>
          </div>
        }
      </section>

      <!-- Actions -->
      <div class="form-actions">
        <button type="button" (click)="saveDraft()" [disabled]="isSaving()">
          Save Draft
        </button>
        <button type="submit" [disabled]="form.invalid || isSaving()">
          Submit
        </button>
      </div>
    </form>
  `,
})
export class DiagnosticCentreComponent implements OnInit {
  private readonly formService = inject(DiagnosticFormService);
  private readonly diagnosticService = inject(DiagnosticService);

  form!: FormGroup;
  isSaving = signal(false);

  performanceMetrics = [
    { key: 'profitability', label: 'Profitability' },
    { key: 'productivity', label: 'Productivity' },
    { key: 'efficiency', label: 'Efficiency' },
    { key: 'customerSatisfaction', label: 'Customer Satisfaction' },
    { key: 'marketResilience', label: 'Market Resilience' },
    { key: 'socialImpact', label: 'Social Impact' },
    { key: 'talentAttraction', label: 'Talent Attraction' },
    { key: 'talentRetention', label: 'Talent Retention' },
  ];

  ngOnInit(): void {
    this.form = this.formService.createForm();
    this.loadExistingData();
  }

  // Form Arrays
  get employeeHashtags(): FormArray {
    return this.form.get('companyInfo.employeeHashtags') as FormArray;
  }

  get strategicPillars(): FormArray {
    return this.form.get('evp.strategicPillars') as FormArray;
  }

  addHashtag(): void {
    this.employeeHashtags.push(this.formService['fb'].control(''));
  }

  removeHashtag(index: number): void {
    this.employeeHashtags.removeAt(index);
  }

  addStrategicPillar(): void {
    this.strategicPillars.push(this.formService['fb'].control(''));
  }

  removeStrategicPillar(index: number): void {
    this.strategicPillars.removeAt(index);
  }

  async loadExistingData(): Promise<void> {
    const data = await this.diagnosticService.getExistingData();
    if (data) {
      this.form.patchValue(data);
      // Populate arrays
      data.companyInfo?.employeeHashtags?.forEach((h: string) => this.addHashtag());
    }
  }

  async saveDraft(): Promise<void> {
    this.isSaving.set(true);
    try {
      await this.diagnosticService.saveDraft(this.form.value);
    } finally {
      this.isSaving.set(false);
    }
  }

  async onSubmit(): Promise<void> {
    if (this.form.invalid) return;

    this.isSaving.set(true);
    try {
      await this.diagnosticService.submit(this.form.value);
    } finally {
      this.isSaving.set(false);
    }
  }
}
```

## Brand Kit Form with Custom Validators

```typescript
// src/app/features/brand-centre/brand-kit.form.ts
import { Injectable, inject } from '@angular/core';
import { FormBuilder, FormGroup, FormArray, Validators, AbstractControl, ValidationErrors } from '@angular/forms';

// Custom validator for hex colour
export function hexColourValidator(control: AbstractControl): ValidationErrors | null {
  const value = control.value;
  if (!value) return null;
  
  const hexPattern = /^#[0-9A-Fa-f]{6}$/;
  return hexPattern.test(value) ? null : { invalidHexColour: true };
}

// Custom validator for UK URL
export function ukUrlValidator(control: AbstractControl): ValidationErrors | null {
  const value = control.value;
  if (!value) return null;
  
  try {
    const url = new URL(value);
    return url.protocol === 'https:' ? null : { httpsRequired: true };
  } catch {
    return { invalidUrl: true };
  }
}

@Injectable({ providedIn: 'root' })
export class BrandKitFormService {
  private readonly fb = inject(FormBuilder);

  createForm(): FormGroup {
    return this.fb.group({
      websiteUrl: ['', [Validators.required, ukUrlValidator]],
      
      colours: this.fb.array([
        this.createColourGroup(),
      ]),
      
      fonts: this.fb.group({
        primary: ['', Validators.required],
        secondary: [''],
        isGoogleFont: [false],
      }),
      
      logos: this.fb.group({
        primary: [''],
        secondary: [''],
        icon: [''],
        monochrome: [''],
      }),
    });
  }

  createColourGroup(): FormGroup {
    return this.fb.group({
      name: ['', Validators.required],
      hex: ['#000000', [Validators.required, hexColourValidator]],
      usage: [''],
    });
  }

  addColour(form: FormGroup): void {
    const colours = form.get('colours') as FormArray;
    colours.push(this.createColourGroup());
  }

  removeColour(form: FormGroup, index: number): void {
    const colours = form.get('colours') as FormArray;
    if (colours.length > 1) {
      colours.removeAt(index);
    }
  }
}
```

## Form State Persistence

```typescript
// src/app/core/services/form-persistence.service.ts
import { Injectable } from '@angular/core';
import { FormGroup } from '@angular/forms';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class FormPersistenceService {
  private readonly STORAGE_PREFIX = 'gptw_form_';

  enableAutosave(form: FormGroup, formKey: string): void {
    form.valueChanges
      .pipe(
        debounceTime(1000),
        distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
      )
      .subscribe(value => {
        this.saveToStorage(formKey, value);
      });
  }

  saveToStorage(formKey: string, value: unknown): void {
    const key = this.STORAGE_PREFIX + formKey;
    sessionStorage.setItem(key, JSON.stringify(value));
  }

  loadFromStorage<T>(formKey: string): T | null {
    const key = this.STORAGE_PREFIX + formKey;
    const stored = sessionStorage.getItem(key);
    return stored ? JSON.parse(stored) : null;
  }

  clearStorage(formKey: string): void {
    const key = this.STORAGE_PREFIX + formKey;
    sessionStorage.removeItem(key);
  }
}
```

## References

- [Angular Reactive Forms](https://angular.dev/guide/forms/reactive-forms)
- [Form Validation](https://angular.dev/guide/forms/form-validation)
- [Dynamic Forms](https://angular.dev/guide/forms/dynamic-forms)
