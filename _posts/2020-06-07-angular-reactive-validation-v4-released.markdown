---
layout: post
title:  "Angular Reactive Validation v4 released"
date:   2020-06-07 08:46:08 +0100
categories: angular
---

Today I released [`angular-reactive-validation`](https://www.npmjs.com/package/angular-reactive-validation) v4.0.1. There were quite a number of breaking changes with Angular 9 Ivy that prevented users of v3.x to upgrade. However, the library is now fully compatible with Angular 9.

<!--more-->

For those of you that are not yet aware of what `angular-reactive-validation` is: it makes adding validation messages with reactive forms far less convoluted. Instead of writing a lot of HTML elements that show and hide themselves based on validity of a `FormControl`:

{% highlight html %}
<form [formGroup]="form">
  <div formGroupName="name">
    <label>First name:
      <input formControlName="firstName">
    </label>
    <label>Middle name:
      <input formControlName="middleName">
    </label>
    <label>Last name:
      <input formControlName="lastName">
    </label>
    <div *ngIf="firstName.invalid && (name.dirty || name.touched)" class="alert alert-danger">
      <div *ngIf="firstName.errors.required">
        A first name is required
      </div>
      <div *ngIf="firstName.errors.minlength">
        The minimum length is 1
      </div>
      <div *ngIf="firstName.errors.maxlength">
        Maximum length is 50
      </div>
    </div>
    <!-- Repeat for all the other input fields... -->
  </div>
  <input type="submit" />
</form>
{% endhighlight %}

You simply configure `FormGroups` like this:

{% highlight typescript %}
import { Validators } from 'angular-reactive-validation';

...

form = this.fb.group({
  name: this.fb.group({
    firstName: ['', [Validators.required('A first name is required'),
      Validators.minLength(1, minLength => `The minimum length is ${minLength}`),
      Validators.maxLength(50, maxLength => `Maximum length is ${maxLength}`)]],
    middleName: ['', [Validators.maxLength(50, maxLength => `Maximum length is ${maxLength}`)]],
    lastName: ['', [Validators.required('A last name is required'),
      Validators.maxLength(50, maxLength => `Maximum length is ${maxLength}`)]]
  }),
  age: [null, [
    Validators.required('An age is required'),
    Validators.min(0, 'You can\'t be less than zero years old.'),
    Validators.max(150, max => `Can't be more than ${max}`)
  ]]
});
{% endhighlight %}

And then write the following HTML:

{% highlight html %}
<form [formGroup]="form">
  <div formGroupName="name">
    <label>First name:
      <input formControlName="firstName">
    </label>
    <label>Middle name:
      <input formControlName="middleName">
    </label>
    <label>Last name:
      <input formControlName="lastName">
    </label>
    <arv-validation-messages [for]="['firstName', 'middleName', 'lastName']"></arv-validation-messages>
  </div>
  <label>Age:
    <input type="number" formControlName="age">
  </label>
  <arv-validation-messages for="age"></arv-validation-messages>
  <input type="submit" />
</form>
{% endhighlight %}

## Comments

Post your comments [here](https://gist.github.com/davidwalschots/9fb9481a795e7f0f2c85d6f9ffe56019).
