---
title: How to  
domain: campzulu.hashnode.dev
tags: lit, web-components, react, vue, lit, lit-elements, polymer-elements
cover: https://cdn.hashnode.com/res/hashnode/image/unsplash/tjX_sniNzgQ/upload/v1664874789026/pAufQE61k.jpeg?w=1600&h=840&fit=crop&crop=entropy&auto=compress,format&format=webp
publishAs: mikaruch
ignorePost: true
---

In one of our current customer projects we will migrate 30+ java fx apps to 30+ webapps.
Since they should look the same and should have the same components our goal is to create a shared
ui library.
The UI library should guarantee longevity, not that it has to be rewritten in a few years.
Currently, react is the most popular framework there is, however who is to say that it will be in
5-10 years?
![NPM trend graph frontend frameworks](https://cdn.hashnode.com/res/hashnode/image/upload/v1664876706565/k2ZfnZDAI.jpg)
Creating this shared UI lib should allow us or our customer to create webapps in 5-10 years with the
same ui lib.
No matter which framework is popular at the time.

Hence we decided to use Web Components.

### Disclaimer

This approach makes sense when you want to reuse the same components in many different projects.
If it is just for one or two websites it is easier to create a shared ui lib with the used
technology (like react).

# WebComponents

> WebComponents are the web standard for creating reusable custom elements<br/>
> \- [MDN web docs](https://developer.mozilla.org/en-US/docs/Web/Web_Components)

Since standardised APIs do not change anymore, it only makes sense to build upon this web feature.
All modern browsers support WebComponents.
![custom elements support matriy](https://cdn.hashnode.com/res/hashnode/image/upload/v1664877714796/Dew6w8JKC.png)
*Safari does not support a subset of the webcomponent standard - the extension of built-in html
elements.*

## Vanilla vs Stencil vs Lit

Now the question was, should we use vanilla js or a framework to help with the WC.
There are two popular frameworks stenciljs and lit (previously polymer).
Ultimately we decided to go with lit for a few reasons. Lit uses template literals while stencil
uses JSX.
Additionally, stencil is tightly bound to webpack. We all know how slow and cumbersome webpack can
be.
On the other hand there is lit, which is able to use whatever bundler we want. In our case we use
vite.
It is super fast since it uses esbuild under the hood.
With our setup we are able to do proper treeshaking since we create for each component one bundle.
Stencil is able to do this as well it just takes a bit longer ;)

```ts
import { defineConfig } from 'vitest/config';
import { resolve, parse } from 'path';
import { globbySync } from 'globby';

const getComponents = () => {
  const components = globbySync('./src/**/!(*.(stories|test)).ts');
  return components.reduce((obj, path) => {
    return { ...obj, [parse(path).name]: resolve(__dirname, path) };
  }, {});
};

export default defineConfig({
  build: {
    outDir: 'dist',
    sourcemap: true,
    // @ts-ignore
    lib: {
      fileName: '[name]',
      formats: [ 'es' ],
    },
    rollupOptions: {
      input: {
        ...getComponents(),
      },
      external: /^lit/,
    },
  },
  test: {
    environment: 'jsdom',
  },
});

```

The syntax is basically the same, the one major difference is stencil needs to be compiled because
of the JSX.
Lit on the other hand could only be transpiled and it would work out of the box.
Stencil split the html and css into two separate files, while everything is in the class in lit.

#### Stencil Button Component

```typescript jsx
import { Component, h, Prop } from '@stencil/core';

@Component({
  tag: 'custom-button',
  styleUrl: 'button.css',
  shadow: true,
})
export class Button {
  @Prop() titleText: string;
  @Prop() type: 'primary' | 'secondary';

  render() {
    return (
      <button
        title={ this.titleText }
        class={ {
          'btn': true,
          'btn-primary': this.type === 'primary',
          'btn-secondary': this.type === 'secondary'
        } }
      >
        <slot></slot>
      </button>
    );
  }
}
```

#### Lit Button Component

```typescript
import { css, html, LitElement } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('custom-button')
export default class Button extends LitElement {
  static override styles = css`...`;

  @property({ type: String })
  titleText: String = '';

  @property({ type: String })
  type: 'primary' | 'secondary' = 'primary';

  override render() {
    return html`
      <button
        class="btn ${ this.type === 'primary' ? 'btn-primary' : 'btn-secondary' }"
        title="${ this.titleText }"
      >
        <slot></slot>
      </button>
    `;
  }
}
```

As you can see it generally looks the same.
And in the end both achieve the same goal of creating WebComponents in a simpler way.

## Framework support

Since WebComponents is a web standard all popular frameworks support it almost out of box.
See [this website](https://custom-elements-everywhere.com/) to get an up-to-date overview.

Since we create business-grade applications we of course use Typescript.
Not just because it is fancy, but also to ensure type safety.

**Lets see how we can ensure code completion and type checking**

### React

React supports web components out of the box.
While the current react version lack some features like complex props passing,
there is a RFC and an experimental react 19 version which fix that issue.

Since we use typescript lets see how we can enable typescript support for our custom elements.
It is quite simple, the elements need to be declared in the `JSX` namespace linking to the custom
element class.

```ts
declare global {
  namespace JSX {
    interface IntrinsicElements {
      'custom-button': Button;
    }
  }
}
```

Partial is necessary here since Button contains a big variety of functions like render,
which we do not want to pass as props.
A bit of a nicer variant would be to extract the props to an Interface
and then to just use the Interface instead of the class.

```ts
interface ButtonProps {
  titleText: String;
  type: 'primary' | 'secondary';
}

@customElement('custom-button')
export default class Button extends LitElement implements ButtonProps {
  static override styles = css`...`;

  @property({ type: String })
  titleText: String = '';

  @property({ type: String })
  type: 'primary' | 'secondary' = 'primary';

  override render() {
    return html`...`;
  }
}

declare global {
  namespace JSX {
    interface IntrinsicElements {
      'custom-button': ButtonProps;
    }
  }
}
```

#### React 18

So what is the problem with React 18? Well as I mentioned above it cannot pass complex props to the
custom element.
Also since react has a synthetic event system, attaching event listeners for custom events directly
to the element does not work.

```tsx
// does not work
const Button = () => {
  const handleEvent = (e: Event) => {
  };
  return <custom-button type='primary' onmy-custom-event={ handleEvent }>Click me!</custom-button>;
}

// works - but ugly
const Button = () => {
  const ref = useRef(null);
  useEffect(() => {
    const handler = () => {
    };
    ref.current.addEventListener('my-custom-event', handler);
    return ref.current.removeEventListener('my-custom-event', handler);
  }, []);
  return <custom-button type='primary' ref={ ref }>Click me!</custom-button>;
}
```

I think we can all agree that this workaround is hurting our eyes and writing it once is already once
too much.
The lit team thought so too and created the `@lit-labs/react` package.
It provides a utility wrapper which creates a React component wrapper for a custom element.

```ts
import type { EventName } from '@lit-labs/react';

import * as React from 'react';
import { createComponent } from '@lit-labs/react';
import Button from './button.ts';

export const CustomButton = createComponent(
  React,
  'custom-button',
  Button,
  {
    onMyCustomEvent: 'my-custom-event' as EventName<MyCustomEvent>
  }
);
```

The CustomButton component then can be used like a normal React Component.

```tsx
<CustomButton onMyCustomEvent={ (e: MyCustomEvent) => console.log("hello") } type='primary'>
  Click me!
</CustomButton>
```

However we are developers, we do not like duplicate code.
The lit team also has a solution for that (unfortunately it does not work for our use case).
The `@lit-labs/cli` has a generation command, which automatically creates a react wrapper package. 

`npx lit labs gen --framework react --outDir ../WebComponentsReact`

This tool heavily relies on the typescript compiler and since we do not use `tsc` to compile our code
the tool unfortunately breaks.
But I am still a lazy developer, this is why I wrote a vite plugin to do the generation for me.
See [vite-plugin-generate-lit-react-wrapper](https://github.com/Cybertron1/vite-plugin-generate-lit-react-wrapper).
The plugin builds upon the `@custom-elements-manifest/analyzer` package, which was developed by Open Web Components.


```ts
import 'virtual:web-components-react-bindings';
```

```ts
import { defineConfig } from 'vitest/config';
import vitePluginCreateLitReactWrapper from '@glytch/vite-plugin-generate-lit-react-wrapper';

export default defineConfig({
  build: {
    outDir: 'dist',
    sourcemap: true,
    lib: {
      entry: './index.ts',
      fileName: '[name]',
      formats: ['es'],
    },
    rollupOptions: {
      external: [/^lit/, 'react', /^web-components/],
    },
  },
  plugins: [
    vitePluginCreateLitReactWrapper({
      globToLitComponents: '../WebComponents/src/**/!(*.spec).ts',
      componentPrefix: 'custom-',
      getComponentPath: (name: string) => `web-components/dist/${name}/${name}`,
      watchLitDist: '../WebComponents/dist',
      outDir: './dist',
    }),
  ],
  test: {
    environment: 'jsdom',
  },
});
```


#### React 19

```ts
type CustomElement<T> = T & { children?: any };

declare global {
  namespace JSX {
    interface IntrinsicElements {
      'custom-button': CustomElement<ButtonProps> & { 'onmy-custom-event': (event: CustomEvent) => void };
    }
  }
}
```

### Svelte

For svelte it is the same story as for react.
Adding the custom element to the `svelte.JSX` namespace enables svelte to understand the component
and type check it accordingly.

```ts
declare global {
  namespace svelte.JSX {
    interface IntrinsicElements {
      'custom-button': Partial<Button>;
    }
  }
}
```

### Vue & Angular

web-types.json
everything works fine but no intelisense and type checking

```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [ vue({
    template: {
      compilerOptions: {
        // treat all tags with a dash as custom elements
        isCustomElement: (tag) => tag.includes('custom-'),
      },
    },
  }) ],
});

```

## Code Completion

react & svelte work with typescript typings
web-types
https://github.com/microsoft/vscode-custom-data

## Conclusion

- conclusion
    - lit is cool
    - the components can be used everywhere
    - small overhead