# Flexible Liquid Font
Helper script to automatically adjust the font size in Liquid templates.

The setup makes sure that one or more variable texts don't overflow in width and instead get their font-size adjusted.
This is especially helpful for On-Site Badges, since they don't support JavaScript.

<table>
  <tr>
    <td>
      <img height="300" alt="image" src="https://github.com/user-attachments/assets/2426aa82-fe01-411e-a7dd-0fb2b976768f" />
      <br>
      Normal name length
    </td>
    <td>
      <img height="300" alt="image" src="https://github.com/user-attachments/assets/041307f2-e24f-4a89-97aa-20e043993d11" />
      <br>
      Extraordinary long name
    </td>
  </tr>
</table>


# Setup
## Step 1: Create template
Create your HTML+Liquid template as usual. If any, upload all used fonts to AWS.

For best results, first write your badge code in a `.html` file locally, preview it in a latest browser like Chrome or Firefox (don't use Internet Explorer or Edge),
then later copy-paste the whole code in the HTML editor.

Make sure that you use following CSS rules for your badge:
```css
html, body {
  margin: 0;
  padding: 0;
}

@page {
  height: 10.0cm; /* badge height in cm/mm/in */
  width: 10.0cm /* badge width in cm/mm/in */
}

#container {
  /* use the exact same sizes as defined in the @page section */
  height: 10.0cm;
  width: 10.0cm;
}
```
... and structure your HTML like this:
```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Badge</title>
        <style>
          /* CSS rules */
        </style>
    </head>
    <body>
        <div id="container">
          <!-- the rest of the badge code goes in here -->
        </div>
    </body>
</html>
```
Even though most tags like `<html>` or `<body>` are going to be removed inside the on-site badge, this ensures a correct preview of the badge in the browser.

## Step 2: Note maximum width
Preview the final badge in the browser. Using developer tools (Inspect Element), find out the maximum allowed width of the text in pixels.
Write that value down as `maxWidth`, this will be needed later. Ensure that there is enough padding around the text:
Due to the nature of the script, some rounding errors might occur and the text might become a tiny bit bigger than the noted value.

## Step 3: Prepare for script execution
Add a temporary block inside your badge's HTML:
```html
<span id="font-test"></span>
```
Also add a temporary CSS rule for the `#font-test` selector to match the font of your variable text.
For example it could look like that:
```css
#font-test {
  font-family: "MyAwesomeFont";
  font-size: 8.93mm;
}
```

## Step 4: Run the script
Copy the following code:
```js
const element = document.getElementById("font-test");
const special = "ßẞ .,-_*'/()";
const l = "abcdefghijklmnopqrstuvxyzäöü";
const alpha = l + l.toUpperCase() + special;

element.innerHTML = "||";
const baseWidth = element.getBoundingClientRect().width;

const lookup = { };
let defaultWidth = 0;
for (const char of alpha) {
    element.innerHTML = "|" + char + "|";
    const width = element.getBoundingClientRect().width - baseWidth;
    const strWidth = width.toFixed(2);
    if (!(strWidth in lookup)) {
        lookup[strWidth] = "";
    }
    lookup[strWidth] += char;
    defaultWidth += width;
}
defaultWidth = (defaultWidth / alpha.length).toFixed(2);

function condition(letters) {
  if (letters.length > 1) {
    return `"${letters}" contains l`;
  } else {
    return `"${letters}"==l`;
  }
}

const [fkey, ...keys] = Object.keys(lookup);
let liquidFont = `{%if ${condition(lookup[fkey])}%}{%assign p=${fkey}%}`;
for (const key of keys) {
    liquidFont += `{%elsif ${condition(lookup[key])}%}{%assign p=${key}%}`;
}
liquidFont += `{%else%}{%assign p=${defaultWidth}%}{%endif%}`;

console.log(`
{% assign maxWidth= ... %}
{% assign words = null | concat: ... | concat: ... %}
/* Start: Auto generated font adjustment */
{%assign maxSize=0%}{%for w in words%}{%assign c=0%}{%assign s=w|split:""%}{%for l in s%}${liquidFont}{%assign c=c|plus:p%}{%endfor%}{%if c>maxSize%}{%assign maxSize=c%}{%endif%}{%endfor%}{%assign factor=1%}{%if maxSize>maxWidth%}{%assign factor=maxWidth|divided_by:maxSize%}{%endif%}
/* End */
`.trim());
```
Paste it in your browser's console while previewing the badge and run it by pressing <kbd>Enter</kbd>.
Don't forget to refresh the browser window after adding the temporary contents from Step 3.
This code runs over the most common possible characters and calculates their width client-side.
If you know of any other characters that could potentially appear inside the variable text, add them into the `special` variable.

The code outputs a very big Liquid snippet that would go over every character present in the variables and sum up their total width.
If the snippet encounters a character, that is not present in the generated lookup loop, it will take the average width.

## Step 5: Clean up and integrate
Remove the temporary content from Step 3. from your badge's HTML code.
Then, inside your `<style>` tag, paste the snippet generated in Step 4.

You will see a couple of `...` placeholders in the beginning of the snippet.
* Replace the `{% assign maxWidth = ... %}` placeholder with the `maxWidth` value from Step 2.
* Replace the `{% assign words = null | concat: ... | concat: ... %}` placeholder(s) with the variable text

The final code could look something like this:
```html
<style>
  /* ... */

  {% assign maxWidth = 510 %}
  {% assign words = null | concat: attendees[0].first_name | concat: attendees[0].last_name %}
  /* the rest of the snippet */

  /* ... */
</style>
```
or like this:
```html
<style>
  /* ... */

  {% assign line = attendees[0].title | append: " " | append: attendees[0].first_name | append: " " | append: attendees[0].last_name %}
  {% assign maxWidth = 248 %}
  {% assign words = null | concat: line %}
  /* the rest of the snippet */

  /* ... */
</style>
```

## Step 6: Finalize
Create a final rule **after** the snippet for your text(s), like this:
```css
.flexible-font {
  font-family: "MyAwesomeFont";
  font-size: calc(8.93mm * {{factor}});
}
```
And use it on your HTML like this:
```html
  <div class="first_name flexible-font">
      {{attendees[0].first_name}}
  </div>
  <div class="last_name flexible-font">
      {{attendees[0].last_name}}
  </div>
```
