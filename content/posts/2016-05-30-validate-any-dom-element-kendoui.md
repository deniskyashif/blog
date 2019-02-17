---
title: "Validate any DOM element with the KendoUI validator"
date: 2016-05-30
draft: false
tags: ["javascript", "kendoui" ]
summary: "Not only inputs."
---

KendoUI's validator allows declaring rules and easily performing client-side validation for input and textarea elements. It offers great flexibility even when it comes to complex cases, when for example input fields are interconnected. Suppose we want to validate an element other than input or textarea, then calling ```validate()``` won't be enough.  

## The Problem

Let's have a form consisting of a textarea and a grid.

```html
<form id="form">
  <textarea name="text">
    Our world is a magical smoke screen
  </textarea>
  <div id="grid"></div>
  <button id="validate">Validate</button>
</form>
```
  
We want to ensure the textarea's value is not longer than 20 characters and the grid contains at least two rows. To do so, we declare our validation rules in the following way:  

```javascript
var validator = $('#form').kendoValidator({
  messages: {
    text: 'Field must contain at most 20 symbols.',
    grid: 'The grid must contain at least two rows.'
  },
  rules: {
    text: function(item) { 
      if (item.is('[name=text]')) {
        return item.val().length < 20;
      }
      return true;
    },
    grid: function(item) {
      if (item.is('[data-role=grid]')) {
        var grid = item.getKendoGrid();
        return grid.dataItems().length > 1;
      }
      return true;
    }
  }
}).data('kendoValidator');
```

Calling ```validator.validate()``` however checks only the textarea element. So I decided to peek in the KendoUI internals to get a better understanding of how the validator works. You can check the code [here](https://github.com/telerik/kendo-ui-core/blob/25e8c5c5ccd176268e8ec9b2dad3722642898970/src/kendo.validator.js#L279).
  
**TL;DR:** The validate function traverses all the input fields, apart from _"submit"_, _"reset"_, _"button"_, _"disabled"_ and _"readonly"_ types and applies the validation rules to them. No wonder why the grid gets omitted.

## ```validateInput``` to the rescue

This function lets us specify explicitly to which element we want to apply our rules. There is no type constraint whatsoever(although it has "input" in its name). We can also see that ```validate``` internally delegates the validation logic for each separate field by passing it to ```validateInput```. Therefore we have to add the following line to our validation code.

```javascript
var isValid = validator.validate() && validator.validateInput('#grid');
```
