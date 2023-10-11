---
title: My first contact with ESM and CJS
date: "2023-10-11T14:58:00.000Z"
description: Why it's hard to work with frontend
---

I've been doing frontend work for the last 3 months. This is the first
time I'm managing to be successful with frontend work. For the last
14 years I focused primarily on backend because I find frontend really
hard. Today I want to share a firsthand experience on why this is
the case for me.


#### The Error

Let's start with the error message.

```
ReferenceError: Vue is not defined
at [...]/node_modules/@highlightjs/vue-plugin/dist/highlightjs-vue.min.js:3:1171
at ModuleJob.run (node:internal/modules/esm/module_job:194:25)
```

Extremely generic issue. HighlightJS Vue Plugin is attempting to use
what appears to be a global Vue variable, which is not defined.
Search for this on the internet may lead you to some info about how
Vue 2 and Vue 3 differ going from a global scope to a local scope.
But the version of HighlightJS I'm using is compatible with Vue 3.

> "@highlightjs/vue-plugin": "^2.1.0"

#### The When

Let's get more context. This error is happening when I run a code
like the following:


```ts
describe('SomeFormPage', () => {
  it('cancels the form and return blablabla', async () => {

    const wrapper = mount(SomeFormPage, {
      props: {},
      attachTo: document.body,
    });
  });
});
```

As soon as I run this test, I get the Vue not defined error.

#### The Component

The relevant part of the component is displayed below:

```vue
<template>
  <component :is="HighlightCode" language="json" :code="myJsonCode.value" />
</template>


<script setup lang="ts">
import {HighlightCode} from "@packages/Bootstrap/highlight";
const myJsonCode = '{"some": "json"}';
</script>
```

highlight.ts
```ts
import 'highlight.js/styles/intellij-light.css';
import hljs from 'highlight.js/lib/core';
import json from 'highlight.js/lib/languages/json';
import hljsVuePlugin from "@highlightjs/vue-plugin";

hljs.registerLanguage('json', json);

const HighlightCode = hljsVuePlugin.component;

export {hljsVuePlugin, HighlightCode}
```

#### The Investigation

One may argue that I'm not registering the component through `Vue.use()`.
That didn't make any difference. I tried using that approach and the outcome
is exactly the same.

One very important information: it works on the browser. If I open
my form, HighlightJS is working perfectly as intended. No error, no
weird behaviors, nothing. So I started a question on trying to figure
out what are the differences between opening the browser and running
the test.

I stumbled upon this link: https://stackoverflow.com/questions/72587750/referenceerror-vue-is-not-defined-vuejs3-jest-testing-library-vue-and-jest.
The issue quite match what I was facing, but the solution involved some
Jest configuration. I'm using Vitest. I spent about 1 hour trying
to figure out if I could replicate that configuration on Vitest
or if it had soemthing to do with `jsdom`, but no luck on that
front.

#### [Appendix] The side quest problem 

As a good PHP developer, I decided to put some `console.log()` inside
the node_modules folder where the error was happening. Unfortunately
JS minified files makes this harder than with PHP and the `vendor`
folder. No problem, we can do this. Turns out I spent quite a 
significant amount of time not understanding why I couldn't see my
`console.log` message. At some point I thought that there was something
in the Javascript community that prevented you from modifying
vendor folders. Really.

Anyway, it turned out to be an issue with PHPStorm Test Runner. 
PHPStorm was not displaying any `console.log()` from files inside
node_modules, but it would display if it came from project-owned files.
Once I figured this out, I switched to a terminal with `npn run test`
so that I could carry on.

#### The Divergence

I was seeing my `console.log()` on my terminal running the test and
then giving me the error. I was not seeing my `console.log` when
running inside the Browser. And as I mentioned, in the browser everything
works. This is when I noticed two distinct files: 
`highlightjs-vue.esm.min.js` and `highlightjs-vue.min.js`.

My first instinct was to copy all contents of `highlightjs-vue.esm.min.js`
into it's CJS counterpart and run the tests and to my surprise, it works.

#### Why?

The thing I liked the most about Vite was that it was created by
the same guy that created Vue. This is something I think is
missing a lot in the frontend community. Vue + Vite + Vitest are
all under a single umbrella and play super nice with each other.
But suddenly I'm running Vite on the browser with ESM and Vitest
on the terminal with CJS. Apparently there is a reason for this
(https://github.com/vitest-dev/vitest/discussions/4233) which I'm
not well versed in the Javascript world enough yet to understand.
But it boils down to a development tool having a discrepancy between
how it runs production code and how it runs test code. It's a sad thing.

#### The Ecosystem

My ability to succeed at building a frontend project strongly suggests
that the frontend ecosystem is improving a lot. I think a big reason
for this is because Frontend is now its own thing. My rationale is that
if you step 10~20 years back, you'll be in a world where we build
applications and the UI is an afterthought. jQuery popularity was
a fight against different browser APIs and the lack of basic functionality
in Javascript. But even after that, Frontend was still where we all
meet. And by "we", I mean backend technologies. It doesn't matter
if you write Java, PHP, Go, C#, ASP.NET, Kotlin, Python, Ruby or Node.
The browser is still the same and the language running in the browser
is still the same. Several communities merge together, each of them
with their own ideas of frontend development.

Nowadays, after the rise of React, Vue, Angular and Svelte, Frontend
development is its own individual thing, community and space. We
have Frontend developers, frontend practices and we're not trying
to simply port a Dependency Injection container into Javascript to
make angularjs (Angular 1). Jest and Vitest don't need a DI container,
they have hoisting capabilities for mocking. 

Bun is a perfect demonstration of what Javascript is missing: a
unified front. Bun is a Package Manager, a Test Runner, a Runtime.
A few years ago if you wanted to write tests, you had to figure out
how to configure Typescript to play nice with Jest, esbuild and 
webpack. Now Deno supports TS files natively.

Instead of trying to stitch together development practices from every
backend programming language in the world, frontend is building
its own solid practices and it's only getting better. It will take
a couple of years to iron out some details and an extra half-decade
to phase out a lot of legacy practices, but the future is looking
good!

#### The Fix

On my `vite.config.js`, I added the following aliases: 

```json
resolve: {
    alias: {
        'vue': 'vue/dist/vue.esm-bundler.js', 
        '@highlightjs/vue-plugin':'@highlightjs/vue-plugin/dist/highlightjs-vue.esm.min.js',
        '@tinymce/tinymce-vue': '@tinymce/tinymce-vue/lib/es2015/main/ts/index.js',
  }
}
```

This forces Vitest to load the ESM version of the library. Since my
frontend skills are **questionable** at best, I have no idea what
are the consequences of this setup. So far it has worked and hasn't
presented any side effects. Time will tell.