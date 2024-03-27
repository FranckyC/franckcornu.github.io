---
title: "Use TailwindCSS with SPFx"
date: 2024-03-27T00:00:00.000Z
# post thumb
images:
 - images/post/using-tailwindcss-with-spfx/tailwind_post.jpg
#author
author: "Franck Cornu"
# description
description: "Demonstrates how to configure and use TailwindCSS in a SPFx project"
# Taxonomies
categories: ["spfx"]
tags: [post]
type: "featured" # available type (regular or featured)
draft: false
sidebar: right
showimage: false
---

-----------------------------

<div style="display:flex;justify-content:space-around;align-items:center">
  <div>
    <image src="https://github.githubassets.com/assets/GitHub-Mark-ea2971cee799.png" style="margin:0" caption="" alt="alter-text"  width="100" />
  </div>
  <div>
    The GitHub solution showcasing the integration is available here: <a target="_blank" href="https://github.com/FranckyC/spfx-tailwind-sample/tree/develop">https://github.com/FranckyC/spfx-tailwind-sample/tree/develop</a>
  </div>
</div>

-----------------------------

To handle CSS customizations, the SharePoint framework comes with a [prebuilt mechanism in its toolchain](https://learn.microsoft.com/en-us/sharepoint/dev/spfx/css-recommendation). Basically, all SASS or CSS stylesheets can be treated as TypeScript modules (ex: `mycomponent.module.scss`). This way, you can directly reference classes in your components using a `styles` variable imported from the generated `.ts` module:

```typescript
import styles from './todoList.module.scss';
// ...

export default class TodoList extends React.Component<ITodoListProps, void> {
  public render(): React.ReactElement<ITodoListProps> {
    return (
      <div className={styles.todoList}>
        <div className={styles.text}>Hello world</div>
      </div>
    );
  }
}
```

Behind the scenes, SPFx generates unique names for CSS classes to avoid name conflitcs with other components on the page and also handle CSS vendors specifities (ex: IE10/11, Safari, etc) for you.

**So what's the problem?**

For simple solutions wih few components and stylesheets, this mechanism is easy to use and works well. However, for more complex solutions including many components and sub components, this approach has some limits.

As an SPFx developer, writing CSS is not my primary skill and not really a passion (and I'm probably not the only one there). Often I get overwhelmed by SCSS stylesheets sprawling in my project structure as long as my solution is evolving. I often need to reorganize, delete or update parts of SCSS stylesheets. I also don't talk about hours I've spent fixing dummy CSS issues...

When writing CSS, you always have to think about containers hierachy, styles reusability, shared variables, etc. According to your components structure and complexity (ex: if your components needs to support theme), this is a time consuming and a cumbersome process as there is no easy way to keep in sync CSS classes usages and declarations in the code. Often, some CSS code parts aren't actually used in my solution due to these multiple changes increasing bundle size and I always feel my styles could have been organized better.

> As an example, I'm pretty sure many CSS classes declared in the PnP Modern Search are not actually used in the code! Anyway..

**But what if you could speed up the CSS writing process, or better, completely getting rid of these numerous stylesheets and avoid writing a single line of CSS code?** 

A dream for every developer no ? 

Well let's talk about [TailwindCSS](https://tailwindcss.com).

## What is TailwindCSS?

{{< image src="images/post/using-tailwindcss-with-spfx/tailwindcss_logo.jpg" caption="" alt="alter-text" position="center" class="img-fluid" title="image title"  webp="false" >}}

From the offical webiste, [TailwindCSS](https://tailwindcss.com/) is a utility-first CSS framework packed with predefined classes that can be composed to build any design, directly in your markup. It works only at **build** time.

**How it works?**

During the build time, the tool scans all files defined according to your configuration (ex: all `.ts`, `.tsx` files specified in the `tailwind.config.js`) looking for well-known classes from the library set and generates the corresponding CSS classes. The stylesheet can then be imported into your components and classes used directly.

**What are the benefits?**

At a glance:

- **No more stylesheets to manage or CSS to write**. You only use well known, predefined CSS classes from the library set. You don't have to worry how it is done behind the scenes in CSS.
- You ship only the CSS you actually use in your code. **No more dead CSS bloating your bundle**.
- Numerous CSS utility classes to build awesome UI in minutes for non CSS specialists (like me). Many examples available from the community ([paid](https://tailwindui.com/) or free depending the complexity).
- Heavily customizable and enough flexibility to cover edge cases. Can be integrated with any framework.
- Super easy light/dark mode support.

## Integration with the SharePoint Framework

When building the [PnP Modern Search Core components](https://github.com/microsoft-search/pnp-modern-search-core-components) and also their SPFx WebParts integration, I decided to give a try to TailwindCSS. In my company, it was already a well known library used by maby internal projects. 

> **I have to say, I probably never go back to traditional CSS/SASS stylesheets anymore. It saved me so much time and allowed me to create such better experiences than can't even think doing the same writing my own CSS**

### Setup TailwindCSS in your SPFx project

First of all, if you work with Visual Studio Code, you can install the [Tailwind CSS IntelliSense extension](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss). Then:

1. Add the TailwindCSS package in your `package.json` file under the `devDependencies` (remember, it works only at build time):

```json
{
 "devDependencies": {
    ...
    "tailwindcss": "3.2.4",
    "@tailwindcss/forms": "0.5.3",
  }
}
```

2. At the root of your solution, create a `tailwind.config.js` file, with this default configuration:

```javascript

const defaultTheme = require('tailwindcss/defaultTheme');

module.exports = {
  mode: 'jit', // allow to update CSS classes automatically when a file is updated (watch mode). See below
  content: {
    files: [
      './src/**/*.{html,ts,tsx}', // scan for these files in the solution
    ]
  },
  corePlugins: {
    preflight: false, // to avoid conflict with base SPFx styles otherwise (ex: buttons background-color)
  },
  darkMode: 'class',
  theme: {
    extend: {
        fontFamily: {
          sans: ['var(--myWebPart-fontPrimary)','Roboto', ...defaultTheme.fontFamily.sans]
        },
        colors: {

          /* light/dark is controlled by the theme values at WebPart level */
          primary: "var(--myWebPart-primary, #7C4DFF)",
          background: "var(--myWebPart-background, #F3F5F6)",
          link: "var(--myWebPart-link, #1E252B)",
          linkHover: "var(--myWebPart-linkHover, #1E252B)",
          bodyText: "var(--myWebPart-bodyText, #1E252B)"
        }
    }
  },
  plugins: [
    require('@tailwindcss/forms'), // to be able to style inputs
  ],
};
```

This file contains basic TailwindCSS customizations for a SPFx project (CSS variables, plugins, etc.). 

3. Create a `styles` folder in your project structure (ex: `/src/styles`) and create the two following files in it:

- `tailwind.css`: this file is used as an entry point for TailwindCSS to generate your styles. It defines all the base classes you want to support.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

```

- `postcss.config.js`: this file is used by [PostCSS](https://postcss.org/) to define what CSS transformations need to be done through plugins (like `tailwindcss` but also handling the vendors prefix support using `autoprefixer` plugin). This is the [preferred way](https://tailwindcss.com/docs/installation/using-postcss) to run TailwindCSS along other build tools.

```javascript
module.exports = {
    plugins: {
      tailwindcss: {},
      autoprefixer: {},
    }
}
```

> You can use any other PostCSS plugins you want here.

4. Update your `package.json` to use following scripts. Also add the package `npm-run-all` to be able to run multiple scripts in parallel:

```json
"scripts": {
    "build": "gulp bundle",
    "build:ship": "gulp bundle --ship",
    "clean": "gulp clean",
    "test": "gulp test",
    "bundle": "npm-run-all taildwindcss build",
    "bundle:ship": "npm-run-all taildwindcss build:ship",
    "webpack:serve": "fast-serve",
    "serve": "npm-run-all -p tailwindcss:watch webpack:serve",
    "taildwindcss":  "tailwindcss -i ./src/styles/tailwind.css -o ./src/styles/dist/tailwind.css --minify --postcss ./src/styles/postcss.config.js",
    "tailwindcss:watch": "tailwindcss -i ./src/styles/tailwind.css -o ./src/styles/dist/tailwind.css --watch --minify --postcss ./src/styles/postcss.config.js"
}
```

```json
{
 "devDependencies": {
    ...
    "npm-run-all": "^4.1.5"
  }
}
```

> I use [SPFx Fast Serve tool](https://github.com/s-KaiNet/spfx-fast-serve) here for convenience

Some explanation here:

- `"taildwindcss":  "tailwindcss -i ./src/styles/tailwind.css -o ./src/styles/dist/tailwind.css --minify --postcss ./src/styles/postcss.config.js"`: generate the CSS stylesheet from the tailwindcss.css file using PostCSS and output the result in a `/dist` folder. **This is this file you need to include in your code**. 
- `"tailwindcss:watch": "tailwindcss -i ./src/styles/tailwind.css -o ./src/styles/dist/tailwind.css --watch --minify --postcss ./src/styles/postcss.config.js"`: same as above but in watch mode using the **Just-in-Time** feature. CSS classes will regenerate when your code is updated.
- `"webpack:serve": "fast-serve"`: just a wrapper script for fast serve.
- `"serve": "npm-run-all -p tailwindcss:watch webpack:serve"`: run the SPFx WebPart build and also TailwindCSS in watch mode using its '[Just-in-time](https://v2.tailwindcss.com/docs/just-in-time-mode)' feature. To get it work with TailwindCSS, the flag  `mode: 'jit'` needs to be set in the `tailwind.config.js` configuration file.
code.
- `"bundle": "npm-run-all taildwindcss build"`: bundle the SPFx solution including TailwindCSS compilation first.

5. Import the stylesheet in your component and use TailwindCSS classes:

```jsx
import '../../styles/dist/tailwind.css';
...

return (
    <section className='overflow-hidden p-1 text-bodyText bg-background font-sans'>
        <div className="text-center">
            <img alt="" src={isDarkTheme ? require('../assets/welcome-dark.png') : require('../assets/welcome-light.png')} className={"w-full max-w-[420px]"} />
            <h2>Well done, {escape(userDisplayName)}!</h2>
            <div>{environmentMessage}</div>
            <div>Web part property value: <strong>{escape(description)}</strong></div>
        </div>
        ...
    </section>

```

> As there is only one generated file and classes are global, it can be imported pretty much anywhere in your code as long it is included in your bundle.

5. Serve your solution using `npm run serve` et voilà!:

{{< image src="images/post/using-tailwindcss-with-spfx/sample_wp.png" caption="" alt="alter-text" position="center" class="img-fluid" title="image title"  webp="false" >}}

## Advanced scenarios

### Deal with the dark mode

In components, you often need to handle both light and dark modes according to the host theme selected (ex: on a SharePoint site). It means CSS classes used has to be aware of what colors from the theme should be applied. Despite TailwindCSS has its own way to handle [dark mode](https://tailwindcss.com/docs/dark-mode), we can't really rely on it with the SharePoint Framework (we could but this is not optimal in my opinion). By default, there is no "link" between TailwindCSS classes and SPFx theme values. **We have to create one**.

The beauty of TailwindCSS is it is very customizable. To create that 'link' we simply extend the default theme definition in the TailwindCSS `tailwind.config.js` configuration file and use custom CSS variables for common colors (ex: body text, primary color, etc.):


```javascript
{
    ...
    theme: {
        extend: {
            fontFamily: {
            sans: ['var(--myWebPart-fontPrimary)','Roboto', ...defaultTheme.fontFamily.sans]
            },
            colors: {
                primary: "var(--myWebPart-primary, #7C4DFF)",
                background: "var(--myWebPart-background, #F3F5F6)",
                link: "var(--myWebPart-link, #1E252B)",
                linkHover: "var(--myWebPart-linkHover, #1E252B)",
                bodyText: "var(--myWebPart-bodyText, #1E252B)"
            }
        }
  } 
}
```

These CSS variables are then set in the component according to the current theme values:

```javascript
// All CSS variables from my project
export enum ThemeCSSVariables {
    fontFamilyPrimary = "--myWebPart-fontPrimary",
    colorPrimary = "--myWebPart-primary",
    background = "--myWebPart-background",
    primaryBackgroundColorDark = "--myWebPart-colorBackgroundDarkPrimary",
    bodyText= "--myWebPart-bodyText",
    link = "--myWebPart-link",
    linkHover = "--myWebPart-linkHover",
}

...
// In the main WebPart class
protected onThemeChanged(currentTheme: IReadonlyTheme | undefined): void {
    if (!currentTheme) {
      return;
    }

    this._isDarkTheme = !!currentTheme.isInverted;
    const {
      semanticColors
    } = currentTheme;

    if (semanticColors) {

        // Map colors from the theme to CSS variables and therefore to TailwindCSS custom colors so they can be used in classes
        this.domElement.style.setProperty(ThemeCSSVariables.colorPrimary, currentTheme?.palette?.themePrimary || null);
        this.domElement.style.setProperty(ThemeCSSVariables.fontFamilyPrimary, currentTheme?.fonts?.medium?.fontFamily || null);
        this.domElement.style.setProperty(ThemeCSSVariables.bodyText, semanticColors.bodyText || null);
        this.domElement.style.setProperty(ThemeCSSVariables.link, semanticColors.link || null);
        this.domElement.style.setProperty(ThemeCSSVariables.linkHover, semanticColors.linkHovered || null);
        this.domElement.style.setProperty(ThemeCSSVariables.background, semanticColors.bodyBackground || null); 
    }
}
```

Then in components you can use the custom colors like this:

```tsx
<section className='text-bodyText bg-background font-sans'>
    ...
</section>
```

## Is there any drawbacks to use TailwindCSS with SPFx?

Unfortunately yes, nothing is perfect! These are more considerations to have when you choose to use TailwindCSS over traditional SASS/CSS stylesheets. Among them:

- As TailwindCSS generates a big CSS stylesheet in the end, there is no more stylesheets isolation (ex: one stylesheet per component as best practice). It can be harder to quickly identify which component uses what classes if you defined a lot of custom classes via theme extentsions.
- If you have multiple Web Parts or extensions on the same page built from different projects/solutions independently, you will end up with duplicated declarations for CSS classes as each solution will generate its own stylesheet. It is not that critical but not optimal regarding the bundle sizes. Also there is no really a risk of name conflicts as class names are standardized. The only conflicts you could have is if other solutions defined the same custom extensions. To mitigate this point:
    - Try to build Web Parts or extensions from the same project using the same TailwindCSS configuration file.
    - Avoid using different version of TailwindCSS across solutions.

## Conclusion

I hope you found this post useful and you will be curious to give it a try in your solutions. To me, TailwindCSS was a game changer in the way I work with the SharePoint Framework and even beyond. It saved me a lot of time in my projects and also probably improved my mental heath to not being forced to write CSS code anymore :D.

**Be careful: once you start to use it, you can't go back 😁**