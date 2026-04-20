---
name: fabricjs-canvas
description: Fabric.js canvas patterns for image editing and template overlays - text placement, badge positioning, export. Use for Activate social post templates.
---

# Fabric.js Canvas Patterns

## When to Use This Skill

Use this skill when:
- Creating social post template overlays
- Positioning text and badges on images
- Building interactive canvas editors
- Exporting final images for social media

## Setup

### Installation

```bash
npm install fabric
npm install @types/fabric --save-dev
```

### Angular Service

```typescript
// src/app/features/activate/services/canvas.service.ts
import { Injectable } from '@angular/core';
import { fabric } from 'fabric';

export interface TemplateConfig {
  width: number;
  height: number;
  backgroundColor: string;
}

export interface TextOverlay {
  text: string;
  x: number;
  y: number;
  fontSize: number;
  fontFamily: string;
  fill: string;
  fontWeight?: string;
  textAlign?: string;
  maxWidth?: number;
}

export interface ImageOverlay {
  url: string;
  x: number;
  y: number;
  width: number;
  height: number;
}

@Injectable({ providedIn: 'root' })
export class CanvasService {
  private canvas: fabric.Canvas | null = null;

  initialize(canvasId: string, config: TemplateConfig): fabric.Canvas {
    this.canvas = new fabric.Canvas(canvasId, {
      width: config.width,
      height: config.height,
      backgroundColor: config.backgroundColor,
      selection: true,
      preserveObjectStacking: true,
    });

    return this.canvas;
  }

  async setBackgroundImage(imageUrl: string): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!this.canvas) {
        reject(new Error('Canvas not initialized'));
        return;
      }

      fabric.Image.fromURL(imageUrl, (img) => {
        if (!this.canvas) return;

        img.scaleToWidth(this.canvas.width!);
        this.canvas.setBackgroundImage(img, () => {
          this.canvas?.renderAll();
          resolve();
        });
      }, { crossOrigin: 'anonymous' });
    });
  }

  addText(config: TextOverlay): fabric.IText {
    if (!this.canvas) {
      throw new Error('Canvas not initialized');
    }

    const text = new fabric.IText(config.text, {
      left: config.x,
      top: config.y,
      fontSize: config.fontSize,
      fontFamily: config.fontFamily,
      fill: config.fill,
      fontWeight: config.fontWeight || 'normal',
      textAlign: config.textAlign || 'left',
    });

    if (config.maxWidth) {
      text.set({ width: config.maxWidth });
    }

    this.canvas.add(text);
    this.canvas.renderAll();

    return text;
  }

  async addImage(config: ImageOverlay): Promise<fabric.Image> {
    return new Promise((resolve, reject) => {
      if (!this.canvas) {
        reject(new Error('Canvas not initialized'));
        return;
      }

      fabric.Image.fromURL(config.url, (img) => {
        img.set({
          left: config.x,
          top: config.y,
          scaleX: config.width / (img.width || 1),
          scaleY: config.height / (img.height || 1),
        });

        this.canvas!.add(img);
        this.canvas!.renderAll();
        resolve(img);
      }, { crossOrigin: 'anonymous' });
    });
  }

  async addBadge(badgeUrl: string, position: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'): Promise<fabric.Image> {
    if (!this.canvas) {
      throw new Error('Canvas not initialized');
    }

    const padding = 20;
    const badgeSize = 100;
    const canvasWidth = this.canvas.width!;
    const canvasHeight = this.canvas.height!;

    let x = padding;
    let y = padding;

    switch (position) {
      case 'top-right':
        x = canvasWidth - badgeSize - padding;
        break;
      case 'bottom-left':
        y = canvasHeight - badgeSize - padding;
        break;
      case 'bottom-right':
        x = canvasWidth - badgeSize - padding;
        y = canvasHeight - badgeSize - padding;
        break;
    }

    return this.addImage({
      url: badgeUrl,
      x,
      y,
      width: badgeSize,
      height: badgeSize,
    });
  }

  addLogo(logoUrl: string, position: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right', maxWidth: number = 150): Promise<fabric.Image> {
    if (!this.canvas) {
      throw new Error('Canvas not initialized');
    }

    return new Promise((resolve, reject) => {
      const padding = 20;
      const canvasWidth = this.canvas!.width!;
      const canvasHeight = this.canvas!.height!;

      fabric.Image.fromURL(logoUrl, (img) => {
        const scale = maxWidth / (img.width || maxWidth);
        const scaledHeight = (img.height || 0) * scale;

        let x = padding;
        let y = padding;

        switch (position) {
          case 'top-right':
            x = canvasWidth - maxWidth - padding;
            break;
          case 'bottom-left':
            y = canvasHeight - scaledHeight - padding;
            break;
          case 'bottom-right':
            x = canvasWidth - maxWidth - padding;
            y = canvasHeight - scaledHeight - padding;
            break;
        }

        img.set({
          left: x,
          top: y,
          scaleX: scale,
          scaleY: scale,
        });

        this.canvas!.add(img);
        this.canvas!.renderAll();
        resolve(img);
      }, { crossOrigin: 'anonymous' });
    });
  }

  exportAsDataUrl(format: 'png' | 'jpeg' = 'png', quality: number = 1): string {
    if (!this.canvas) {
      throw new Error('Canvas not initialized');
    }

    return this.canvas.toDataURL({
      format,
      quality,
      multiplier: 2, // 2x resolution for retina
    });
  }

  exportAsBlob(format: 'png' | 'jpeg' = 'png', quality: number = 1): Promise<Blob> {
    return new Promise((resolve, reject) => {
      if (!this.canvas) {
        reject(new Error('Canvas not initialized'));
        return;
      }

      this.canvas.getElement().toBlob(
        (blob) => {
          if (blob) {
            resolve(blob);
          } else {
            reject(new Error('Failed to create blob'));
          }
        },
        `image/${format}`,
        quality
      );
    });
  }

  clear(): void {
    this.canvas?.clear();
  }

  destroy(): void {
    this.canvas?.dispose();
    this.canvas = null;
  }
}
```

## Social Post Template Component

```typescript
// src/app/features/activate/social-post-editor/template-editor.component.ts
import { Component, ElementRef, ViewChild, OnInit, OnDestroy, inject, input, output } from '@angular/core';
import { CommonModule } from '@angular/common';
import { CanvasService, TemplateConfig, TextOverlay } from '../services/canvas.service';

interface SocialPostTemplate {
  id: string;
  name: string;
  size: 'story' | 'landscape' | 'square';
  config: TemplateConfig;
}

const TEMPLATE_SIZES: Record<string, TemplateConfig> = {
  story: { width: 1080, height: 1920, backgroundColor: '#ffffff' },
  landscape: { width: 1200, height: 630, backgroundColor: '#ffffff' },
  square: { width: 1080, height: 1080, backgroundColor: '#ffffff' },
};

@Component({
  selector: 'app-template-editor',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="template-editor">
      <!-- Size Selector -->
      <div class="size-selector mb-4">
        <button
          *ngFor="let size of sizes"
          [class.active]="selectedSize() === size"
          (click)="onSizeChange(size)"
          class="px-4 py-2 border rounded mr-2">
          {{ size | titlecase }}
        </button>
      </div>

      <!-- Canvas Container -->
      <div class="canvas-container" [style.maxWidth.px]="previewWidth">
        <canvas #canvasElement id="templateCanvas"></canvas>
      </div>

      <!-- Controls -->
      <div class="controls mt-4 space-y-4">
        <!-- Text Input -->
        <div>
          <label class="block text-sm font-medium mb-1">Headline</label>
          <input
            type="text"
            [value]="headlineText"
            (input)="onHeadlineChange($event)"
            class="w-full px-3 py-2 border rounded"
            placeholder="Enter headline text"
          />
        </div>

        <!-- Badge Position -->
        <div>
          <label class="block text-sm font-medium mb-1">Badge Position</label>
          <select (change)="onBadgePositionChange($event)" class="w-full px-3 py-2 border rounded">
            <option value="top-right">Top Right</option>
            <option value="top-left">Top Left</option>
            <option value="bottom-right">Bottom Right</option>
            <option value="bottom-left">Bottom Left</option>
          </select>
        </div>

        <!-- Export Button -->
        <button
          (click)="exportImage()"
          class="w-full px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700">
          Download Image
        </button>
      </div>
    </div>
  `,
})
export class TemplateEditorComponent implements OnInit, OnDestroy {
  @ViewChild('canvasElement', { static: true }) canvasElement!: ElementRef<HTMLCanvasElement>;

  private readonly canvasService = inject(CanvasService);

  backgroundImage = input<string>();
  badgeUrl = input<string>();
  logoUrl = input<string>();
  brandColour = input<string>('#0066cc');

  imageExported = output<Blob>();

  sizes = ['story', 'landscape', 'square'] as const;
  selectedSize = signal<'story' | 'landscape' | 'square'>('landscape');
  previewWidth = 600;
  headlineText = '';

  private headlineObject: fabric.IText | null = null;

  ngOnInit(): void {
    this.initializeCanvas();
  }

  ngOnDestroy(): void {
    this.canvasService.destroy();
  }

  private async initializeCanvas(): Promise<void> {
    const config = TEMPLATE_SIZES[this.selectedSize()];
    this.canvasService.initialize('templateCanvas', config);

    // Set background if provided
    const bgImage = this.backgroundImage();
    if (bgImage) {
      await this.canvasService.setBackgroundImage(bgImage);
    }

    // Add badge if provided
    const badge = this.badgeUrl();
    if (badge) {
      await this.canvasService.addBadge(badge, 'top-right');
    }

    // Add logo if provided
    const logo = this.logoUrl();
    if (logo) {
      await this.canvasService.addLogo(logo, 'bottom-left');
    }
  }

  onSizeChange(size: 'story' | 'landscape' | 'square'): void {
    this.selectedSize.set(size);
    this.canvasService.clear();
    this.initializeCanvas();
  }

  onHeadlineChange(event: Event): void {
    const text = (event.target as HTMLInputElement).value;
    this.headlineText = text;

    if (this.headlineObject) {
      this.headlineObject.set('text', text);
      this.canvasService['canvas']?.renderAll();
    } else if (text) {
      const config = TEMPLATE_SIZES[this.selectedSize()];
      this.headlineObject = this.canvasService.addText({
        text,
        x: 40,
        y: config.height - 200,
        fontSize: 48,
        fontFamily: 'Arial',
        fill: '#ffffff',
        fontWeight: 'bold',
        maxWidth: config.width - 80,
      });
    }
  }

  onBadgePositionChange(event: Event): void {
    const position = (event.target as HTMLSelectElement).value as 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right';
    // Re-render with new badge position
    this.canvasService.clear();
    this.initializeCanvas();
    const badge = this.badgeUrl();
    if (badge) {
      this.canvasService.addBadge(badge, position);
    }
  }

  async exportImage(): Promise<void> {
    const blob = await this.canvasService.exportAsBlob('png');
    this.imageExported.emit(blob);

    // Also trigger download
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `social-post-${this.selectedSize()}.png`;
    link.click();
    URL.revokeObjectURL(url);
  }
}
```

## Template Presets

```typescript
// src/app/features/activate/data/template-presets.ts
export interface TemplatePreset {
  id: string;
  name: string;
  size: 'story' | 'landscape' | 'square';
  textPosition: { x: number; y: number };
  badgePosition: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right';
  logoPosition: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right';
  overlayColour: string;
  overlayOpacity: number;
}

export const TEMPLATE_PRESETS: TemplatePreset[] = [
  {
    id: 'modern-blue',
    name: 'Modern Blue',
    size: 'landscape',
    textPosition: { x: 40, y: 400 },
    badgePosition: 'top-right',
    logoPosition: 'bottom-left',
    overlayColour: '#0066cc',
    overlayOpacity: 0.7,
  },
  {
    id: 'minimal-white',
    name: 'Minimal White',
    size: 'square',
    textPosition: { x: 40, y: 800 },
    badgePosition: 'top-left',
    logoPosition: 'bottom-right',
    overlayColour: '#ffffff',
    overlayOpacity: 0.9,
  },
  {
    id: 'story-gradient',
    name: 'Story Gradient',
    size: 'story',
    textPosition: { x: 40, y: 1600 },
    badgePosition: 'top-right',
    logoPosition: 'bottom-left',
    overlayColour: '#1a1a2e',
    overlayOpacity: 0.8,
  },
];
```

## References

- [Fabric.js Documentation](http://fabricjs.com/docs/)
- [Fabric.js Demos](http://fabricjs.com/demos/)
