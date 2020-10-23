---
title: Dynamic Validation Made Easy With Stateful Validators
layout: post
author: Justin Miller
---

Angular provides us a very robust form validation system. If you haven't used it yet, try it out! It's easy to create simple forms with basic validation. However, there are some cases where the form behavior may need to change based on what the user has already entered. For instance, we may want the user to add additional info if they select a certain option in a radio group.

{% include video_figure.html
	src="/assets/posts/2020-10-22-dynamic-validation-made-easy-with-stateful-validators/Dynamic Form Validation.mp4"
	caption="Changing the form validation based on user input"
%}

Having functionality like this is surprisingly common. The example above is for a survey question, but there are plenty of other situations where an optional field needs to dynamically be required based on some other value in the form.

So how does the code for this look? There are a couple of ways to go about implementing this behavior. The first way is simple and works great for small forms like the above. It is purely reactive and uses the `valueChanges` observable on the `FormControl` in order to determine when we should change our form configuration.
{% raw %}
```html
<form [formGroup]="form">
  <fieldset>
    <h2>How did you hear about us?</h2>
    <input
      type="radio"
      name="options"
      id="magazine"
      value="magazine"
      formControlName="options"
    />
    <label for="magazine">Magazine</label>
    <br />
    <input
      type="radio"
      name="options"
      id="internet-ad"
      value="ad"
      formControlName="options"
    />
    <label for="internet-ad">Internet Ad</label>
    <br />
    <input
      type="radio"
      name="options"
      id="word-of-mouth"
      value="wordOfMouth"
      formControlName="options"
    />
    <label for="word-of-mouth">Word of mouth</label>
    <br />
    <input
      type="radio"
      name="options"
      id="other"
      value="other"
      formControlName="options"
    />
    <label for="other">Other</label>
    <br />
    <input type="text" formControlName="otherText" />
  </fieldset>
</form>
<br />
Form state: {{ form.valid ? "Valid" : "Invalid" }}
```
{% endraw %}

```typescript
@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"]
})
export class AppComponent {
  form: FormGroup;
  constructor(fb: FormBuilder) {
    this.form = fb.group({
      options: fb.control(""),
      otherText: fb.control("")
    });

    this.otherTextControl.disable();

    this.optionsControl.valueChanges.subscribe(val => {
      if (val === "other") {
        this.otherTextControl.enable();
        this.otherTextControl.setValidators([Validators.required]);
        this.otherTextControl.updateValueAndValidity();
      } else {
        this.otherTextControl.setValidators([]);
        this.otherTextControl.updateValueAndValidity();
        this.otherTextControl.disable();
      }
    });
  }

  get optionsControl() {
    return this.form.get('options');
  }

  get otherTextControl() {
    return this.form.get('otherText');
  }
}
```

As you can see above, we subscribe to the control that represents our radio button group. We change the disabled state and validators of the text control when the value of the radio group is `"other"`. Finally, when we tick any other radio button, we remove the validators and disable the controls. Everything is awesome!

## The Problem
Okay, so we did it. We have the validators update when the user ticks a specific radio button. However, software development is rarely this simple. Projects typically are in a state of flux where requirements are changed as we receive more user feedback.

So let's say we have a new requirement. The validator for the text control needs to be more dynamic. For the sake of this argument, we will require a custom character limit, which is dependent on the value of a separate radio group:

{% raw %}
```html
<fieldset>
    <h2>How many characters can you type?</h2>
    <input
      type="radio"
      name="limit"
      id="limit-1"
      value="1"
      formControlName="limit"
    />
    <label for="limit-1">1</label>
    <br />
    <input
      type="radio"
      name="limit"
      id="limit-5"
      value="5"
      formControlName="limit"
    />
    <label for="limit-5">5</label>
    <br />
    <input
      type="radio"
      name="limit"
      id="limit-10"
      value="10"
      formControlName="limit"
    />
    <label for="limit-10">10</label>
  </fieldset>
```
{% endraw %}

With this, we will need another `valueChanges` subscription to update the validators on the text input. Let's work on that:

```typescript
this.limitControl.valueChanges.subscribe(val => {
  this.otherTextControl.setValidators([Validators.maxLength(val)]);
});
```

Hmm, ok. Now we update the validators for the text control to have a `maxLength` of the radio group value. Let's see how this works out for us:

{% include video_figure.html
	src="/assets/posts/2020-10-22-dynamic-validation-made-easy-with-stateful-validators/Fail 1.mp4"
	caption="Character limit not being applied after ticking \"Other\" radio button"
%}

That didn't work. So what happened? Angular form validators are immutably set. There currently is no way to update the list of validators, so we can't freely add or remove validator functions as we wish. When we call `setValidators` and give it an array of validator functions, Angular combines all of those validator functions into one.


> There is a trick you can use to append a `ValidatorFn` to a form control's existing `ValidatorFn`. In some special cases, this may end up working for you!
>
>```typescript
this.limitControl.valueChanges.subscribe(val => {
  this.otherTextControl.setValidators(
    [this.otherTextControl.validator, Validators.maxLength(val)]
  );
});
>```
>
>However, this approach is dangerous because it is an additive one. You may end up with unpredictable behavior if you end up calling `setValidators` like this multiple times. So typically, it's best to use this only for one-shot validator updates. Of course, this won't fix our current issue because we have no way to remove the individual validators.

So if we can't easily manipulate a control's `ValidatorFn`, what's the solution?

## Stateful ValidatorFn's
Due to the complexity that comes with dynamic validation, it's best to write our logic in a way that scales well. Reacting to changes in form values is good for less complex form behavior, but the code becomes more unreadable as the form requirements change and constraints on the behavior of the form are tightened.

We can solve this problem by creating custom validator functions within the scope of our component! While this may seem daunting at first, we can sigh in relief since the validator functions Angular provides us are usable as first-class functions! So no need to worry about muddying up our code with another `maxLength` implementation.

Another thing that's nice about this approach is it ties our validation logic to the respective form controls. Since the only control that needs validation is the `otherTextControl`, we will only need to write one validator function! Here is what it looks like:

```typescript
otherTextControlValidator(control: AbstractControl): ValidationErrors | null {
  if (!this.form) {
    return null;
  }
  if (this.optionsControl.value === "other") {
    const required = Validators.required(control);
    if (required) {
      return required;
    }
    const maxLength = Validators.maxLength(this.limitControl.value)(control);
    if (maxLength) {
      return maxLength;
    }
    return null;
  }
  return null;
}
```

Let's break this down. We define our function signature to take a form control and return either `ValidationErrors` or `null`. The `ValidationErrors` type is a map representing the validation errors of the control. This can be customized to your liking so you can easily display errors in your template. In our case, we are forwarding the map returned from Angular's built-in validator functions. When we return `null`, we are telling Angular that this validation function passes.

Because this function is defined as a class method, you cannot pass it directly into the form control without binding it to the class object. Here is how it can be applied:

```typescript
otherText: fb.control("", [this.otherTextControlValidator.bind(this)])
```

As you may have noticed, this function is referencing component properties. This is what makes it stateful. We utilize this state to easily select which validation rules we want to apply based on the values in the form. What's nice about this is we don't have to update validation logic when a value changes somewhere else in the form.

## Caveats
Although this is a great way to handle dynamic validation, there are some issues when using it.

#### Controls will not perform validation unless THEIR value is changed
In our example, we create a validator function for a specific form control. The validator function will only be processed when that form control is updated. A problem we see here is that the form control should be validated when we tick a different radio button in the `"limit"` group, but instead, nothing happens. There are a couple of ways to work around this:

1. Subscribe to `valueChanges` on the `"limit"` form control and call `updateValueAndValidity()` on the `optionText` form control.
2. Rewrite the validator with the intent to apply it to the form group itself. This is not recommended since it will do more validation checks than necessary.

#### What about other form control state?
In our example, we have a requirement to enable/disable the `optionText` form control based on the value of the `optionsControl` form control. Where you place this logic is subjective. In our case, we need to subscribe to `valueChanges` on the `optionsControl` form control anyway, so it's best to place that state logic there. However, if you wanted to change the form control state when the value is modified (e.g., telephone number), you have the option to place that logic in your validator function.

My personal opinion on this is to place non-validation-related logic in your validator function only if it is absolutely necessary. It may make the code more readable by doing it this way, so just keep in mind that you have that option available.

## Conclusion
Stateful validators can be really useful when dealing with complex form logic! It solves the issue of form controls only being able to set validators immutably and helps make our code more scalable and readable. Think about using this approach the next time you have to create an Angular form with complicated form validation.

<iframe src="https://codesandbox.io/embed/dynamic-form-validation-k6v96?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fapp%2Fapp.component.ts&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="Dynamic Form Validation"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>
