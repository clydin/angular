---
title: Signal Form with a Custom Form Control
summary: Integrates a custom-built component into a signal form by implementing the `FormValueControl` interface, making it compatible with the `[control]` directive.
keywords:
  - signal forms
  - form
  - control
  - custom form control
  - FormValueControl
  - model
  - submit
required_packages:
  - '@angular/forms'
related_concepts:
  - 'signals'
---

## Purpose

The purpose of this pattern is to create highly reusable form controls with encapsulated UI and logic. It solves the problem of building complex or domain-specific inputs (like a star rating or a rich text editor) that can be seamlessly integrated into any signal form, just like a native HTML element.

## When to Use

Use this pattern when you have a form input that is more complex than a standard HTML element, such as a star rating component, a custom slider, or a rich text editor. By creating a custom form control, you can seamlessly integrate it into your signal forms. This supersedes the `ControlValueAccessor` pattern from `ReactiveFormsModule` for signal-based forms.

## Key Concepts

- **`FormValueControl<T>`:** An interface that custom form control components can implement to integrate with signal forms.
- **`model()`:** A function from `@angular/core` that creates a two-way bound signal input, used here to implement the `value` property of the `FormValueControl` interface.
- **`(ngSubmit)`:** An event binding on a `<form>` element that triggers a method when the form is submitted.

## Example Files

This example consists of two standalone components: a product form and a reusable numeric stepper control.

### numeric-stepper.component.ts

This file defines the custom form control, which implements the `FormValueControl` interface to integrate with signal forms.

```typescript
import { Component, model, ChangeDetectionStrategy } from '@angular/core';
import { FormValueControl } from '@angular/forms/signals';

@Component({
  selector: 'app-numeric-stepper',
  standalone: true,
  template: `
    <button type="button" (click)="decrement()">-</button>
    <span>{{ value() }}</span>
    <button type="button" (click)="increment()">+</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class NumericStepperComponent implements FormValueControl<number> {
  value = model.required<number>();

  increment() {
    this.value.update(v => v + 1);
  }

  decrement() {
    this.value.update(v => v - 1);
  }
}
```

### product-form.component.ts

This file defines the parent form component that consumes the custom numeric stepper.

```typescript
import { Component, signal, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { form } from '@angular/forms/signals';
import { JsonPipe } from '@angular/common';
import { NumericStepperComponent } from './numeric-stepper.component';

export interface ProductForm {
  name: string;
  quantity: number;
}

@Component({
  selector: 'app-product-form',
  standalone: true,
  imports: [JsonPipe, NumericStepperComponent],
  templateUrl: './product-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductFormComponent {
  @Output() submitted = new EventEmitter<ProductForm>();

  productModel = signal<ProductForm>({
    name: 'Angular T-Shirt',
    quantity: 1,
  });

  productForm = form(this.productModel);

  handleSubmit() {
    this.submitted.emit(this.productForm().value());
  }
}
```

### product-form.component.html

This file provides the template for the parent form, showing how the `[control]` directive is used on the custom component.

```html
<form (ngSubmit)="handleSubmit()">
  <div>
    <label>
      Product Name:
      <input type="text" [control]="productForm.name" />
    </label>
  </div>

  <div>
    <label>
      Quantity:
      <app-numeric-stepper [control]="productForm.quantity"></app-numeric-stepper>
    </label>
  </div>

  <button type="submit">Submit</button>
</form>
```

## Usage Notes

- The buttons in the `NumericStepperComponent` have `type="button"` to prevent them from submitting the parent form.
- The `[control]` directive on the `<app-numeric-stepper>` element automatically binds the `productForm.quantity` field to the `value` model of the `NumericStepperComponent`.

## How to Use This Example

The parent component listens for the `(submitted)` event and receives the strongly-typed form data.

```typescript
// in app.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { ProductFormComponent, ProductForm } from './product-form.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductFormComponent, JsonPipe],
  template: `
    <h1>Product Order</h1>
    <app-product-form (submitted)="onFormSubmit($event)"></app-product-form>
    
    @if (submittedData) {
      <h2>Submitted Data:</h2>
      <pre>{{ submittedData | json }}</pre>
    }
  `,
})
export class AppComponent {
  submittedData: ProductForm | null = null;

  onFormSubmit(data: ProductForm) {
    this.submittedData = data;
    console.log('Product data submitted:', data);
  }
}
```
