---
title: Efficiently Displaying Formatted Data
layout: post
author: Justin Miller
---

## The Different Methods

When working in Angular, one issue that comes up often is how to display data that needs to be formatted. This can be done in a number of ways, but deciding which way is best isn't straight forward. The main issue to keep in mind is performance and how expensive it is to format the data. The other thing to keep in mind is change detection and why it matters in this situation.

There are 4 ways to format data before displaying it. They require varying amounts of effort from least to greatest:

* Functions
* Temporary State
* Pipes
* Components

### Functions In The Template
One of the first things we find out when learning how to use Angular is being able to use expressions in our component templates. We can use expressions to construct content in the DOM and even use them when passing data to other components. For displaying data directly, this is usually fine:

{% raw %}
```html
{{ name }}
```
{% endraw %}

For our examples we will use a name, nice and simple. However, the name available is a single string with first and last name combined. Let's say we want to format our name so that we are only displaying the first name. One way we can do this is by using a function within our template. This will work as long as we define it in the component.

{% raw %}
```html
{{ firstName(name) }}
```
{% endraw %}

```typescript
@Component({
	...
})
export class MyComponent {
	@Input() name: 'Bob Lawblaw';
	
	firstName(name) {
		return name.split(' ')[0];
	}
}
```
This works! The component will run the function and render only the first name. This is a very easy way to display our data formatted in the way we would like. However there are some caveats.

#### Change Detection
It is important to keep in mind how often change detection is being triggered. Usually, your components will use one of 2 types of change detection:

* `Default`
* `OnPush`

**`Default`** change detection is eager. It will always check for changes if anything happens. When change detection occurs, Angular will run all function expressions even if the data provided to those functions hasn't changed. It does this to compare against the previous value so that it knows if a change has occured and should be rendered.

{% include video_figure.html
	src="/assets/posts/2020-05-19-displaying-formatted-data-efficiently/Unintended CD Trigger.mp4"
	caption="Changing a non-rendered input causes two CD triggers because we are in developer mode"
%}

For this example, it's not that big of a deal. We are just yanking out the first name from our string. This is not costly at all. But imagine a table of users where every field needs to be formatted in a similar or slightly more costly fasion. With a large number of entries, we can see this becoming an issue of performance where simply scrolling the page can cause jank.

**`OnPush`** change detection more explicit on when change detection occurs. As of Angular v9, it only performs change detection on the component if:

* Input has changed (by reference)
* Component fires an event
* Observable within the component fires an event

This gives us a lot more control over when we want change detection to occur. However, this does not change the fact that Angular will still perform comparison checks on every expression in the template. This means that if our component is handling other data and it changes, our function to get the first name will still be called.

If you are used to the Default change detection strategy and switch to `OnPush`, you may notice that your component may not function like it did before. So be sure to get some practice working with this strategy since it is key to making large Angular apps performant.

More on Change Detection can be read in [this blog post by Angular University](https://blog.angular-university.io/how-does-angular-2-change-detection-really-work/).

### Temporary State
This method focuses on the idea that we only want to perform the formatting function when we know the provided data has changed. We then take the result of that function and store it as a property in our component, like so:

```typescript
@Component({
	...
})
export class MyComponent implements OnChanges {
	@Input() name: 'Bob Lawblaw';
	firstName = '';

	ngOnChanges(changes: SimpleChanges) {
		if (changes.name) {
			this.firstName = this.firstName(changes.name.currentValue);
		}
	}
	
	firstName(name) {
		return name.split(' ')[0];
	}
}
```

We can then reference this property directly rather than calling our function in the template:

{% raw %}
```html
{{ firstName }}
```
{% endraw %}

This prevents change detection from running the function every time because we have removed it from the template and use it only when our data has changed. Although this solves our performance issues, this method is not very developer-friendly since it is not reusable and can build up a lot of clutter in our components. One way to make things a bit cleaner is to use pipes!

### Pipes
Pipes are a great way to abstract your formatting logic! They are built with change detection in mind so that you don't have to think about it in the first place. They also only run when the piped value or pipe parameter is changed by default. So let's make one using the function we had from earlier:

```typescript
@Pipe({ name: 'firstName' })
export class FirstNamePipe implements PipeTransform {
	transform(name: string): string {
		return name.split(' ')[0];
	}
}
```

We can then use it with our name string in the template:

{% raw %}
```html
{{ name | firstName }}
```
{% endraw %}

By doing it this way, we completely isolate the formatting logic, removing it from the component. We also don't have an extra property on our component just to store the calculated value. Nice and clean.

The issue with doing it this way is that it is tedious. For instance, we may be working on a component that has specific formatting logic that won't be used elsewhere in our app. Creating a pipe just for that sounds cumbersome even if it does isolate our formatting logic. [There are proposals](https://github.com/angular/angular/issues/25976) to possibly have a decorator for component functions that would simply treat the function as a pipe which would make more sense for a situation like that.

### Components
This is mostly useful if you are passing a lot of data and need to transform it into something more complex. This is no different than how you normally use components except I urge that a component that is focused on formatting and displaying data should use the `OnPush` change detection strategy. This is to prevent performing change detection on the component until an `Input` is changed.

{% raw %}
```typescript
@Component({
	selector: 'vertical-name'
	template: `<p *ngFor="let char of splitName">{{ char }}</p>`,
	changeDetection: ChangeDetectionStrategy.OnPush
})
export class VerticalNameComponent implements OnChanges {
	@Input() name: 'Bob Lawblaw';
	splitName = [];

	ngOnChanges(changes: SimpleChanges) {
		if (changes.name) {
      		this.splitName = changes.name.currentValue.split('');
    	}
	}
}
```
{% endraw %}

With this component, we can use it within our main component and pass in the name directly:

{% raw %}
```html
<vertical-name [name]="name"></vertical-name>
```
{% endraw %}

Although this is a lot of work for this situation, it is definitely useful to create components like these for complex renders. One example that I have written that follows this approach is the [QR Code Module](https://github.com/SuperiorJT/angular2-qrcode). It is just one component that forwards the input data into a QR Code js library (QRious) to render a QR Code.

## Conclusion
Hopefully this has helped you understand the importance of how we handle formatting data in our templates. With these 4 methods we can see that there needs to be some improvement with how Angular can approach this problem in the future. Ideally there would be one solution that doesn't require having to import something into our NgModule in order to be efficient with the added benefit of keeping our components clean. But for now, my recommendation is as follows:

* If you are just trying to format content inline, try to use `Pipes`. They are pure, easy to test, and avoid a lot of the pitfalls explained above.
* If you need to do some structural rendering or have complex data to format, try creating an `OnPush` `Component`. This will give you all of the control you need in order to render your data. It is also just as reusable as a `Pipe` since all formatting logic is isolated!