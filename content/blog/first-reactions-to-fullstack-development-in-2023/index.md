---
title: "First reactions to fullstack development in 2023"
date: "2023-09-12T23:15:21.003Z"
description:  "Expanding from backend to fullstack"
---

After 6 years working on a 20 years old PHP codebase, I had tammed the backend enough
to have an incredibly productive architecture. I have blogged a bit about this journey.
I've been running Laravel on AWS Lambda since 2019 and I've blogged a bit about it:

- https://blog.deleu.dev/running-lambda-behind-application-load-balancer/
- https://blog.deleu.dev/authenticating-aws-cognito-with-laravel/
- https://blog.deleu.dev/a-monorepo-approach-to-larger-modules-in-laravel-and-lambda/
- https://blog.deleu.dev/what-it-means-to-run-a-monolith-on-aws-lambda/

I've also worked hard on getting logs and metrics for our team into AWS Opensearch.

- https://blog.deleu.dev/el4k-my-journey-through-aws-elk-stack/

I have successfully implemented CI/CD and cross-region deployments:

- https://blog.deleu.dev/cross-region-deployments-with-aws-codepipeline/

And many more.

During these 6 years I have dipped my toes on frontend development a couple of times,
but always left it after getting kicked. I partially attribute this to two things:
my local development setup used to be way more complex than it needed to be and
frontend development used to be way more complex than it needed to be. A combination
that was not great for me. I used to have a Windows PC running VirtualBox with Alpine
Linux with Docker & Git installed. Anything I wanted to run had to run inside a Docker
container in my Alpine Linux sharing source code with my Windows PC. Combine this
with the fragmentation of frontend development 
(webpack, babel, npm, typescript, twitter bootstrap, react, jest, cypress, etc) and it
was a recipe for disaster.


#### A new beginning

About 3 months ago I decided to try renewing my development stack again. I had piled
on a couple of tedious tasks together with some innovation that I had in the back of
my mind. The end goal was to address everything in a single big-bang.

- Monorepo approach between two git repositories I manage.
- Deploy with AWS CDK instead of Serverless/AWS SAM
- Put my entire backend behind a single Domain Name
- Build a fullstack application to simplify frontend/backend development and deployment
- Use InertiaJS
- React, Tailwind, Vite.

React was mostly because it's what the frontend team at the company was using, but
I confess I failed that part a little. I couldn't get Typescript type-check to
properly work with SVG files in React, Vitest was not successfully processing React
components and in general I have a hard time working with React devleopment where
templates contains `${someVariableContent()}`. I know for a fact this is just a me-problem,
none of it was "unsolvable", but I just couldn't accomplish in a short time a great
frontend development architecture that I was happy with. I spoke with the frontenders
and asked if I could try Vue and they were happy to give me a chance. The thing
about Vue that drew me was basically the Laravel community support and the fact that
Vue seems to be somewhat developed with the same mindset that Taylor develops Laravel.
Vue, Vite and Vitest being created by a single umbrella makes it so much easier to
integrate and understand how testing frontend works. You need:

- A Test Runner (vitest)
- A Typescript Builder compatible with your Test Runner (vitest)
- A browser api mock (jsdom)
- An Assertion API (vitest / expect)
- A Vue Testing Library (vue-test-utils)

With PHP and Laravel, you just need PHPUnit and you're ready to go, so I'm not super
used to this kind of setup, but Vue makes it easier by bringing everything under
a similar umbrella. When I think back about trying some of these things back in 2019,
I don't know if ts-jest even existed back then, but I know that getting 
Jest, Babel, Typescript, TS Loader, Webpack, React, etc all to play nice with each other
was not a simple task, specially when working on a 5-year-old React codebase.

#### Inertia

Inertia is a brilliant piece of software. It's not really a framework, it's more of
a small library that implements a concept, and this is the key aspect. It actually
made me reflect back and understand some of the things I always hated in frontend
development.

Frontend is full of state. State is one of the biggest source of computer engineering
nightmare. That's why for 30 years we have been saying "have you tried turning it off
and on again?". It essentially reflects back to the development flow. You write an
invalid syntax, you need to fix it, refresh the page and start from a clean slate.
Every bug you introduce into your code, you refresh it and try from a clean slate.
This process drivers developers to make software that usually work if you follow
the exact state that the developers followed while developing it. By accummulating
a lot of unexpected state, you might find edge-cases that was not discovered by
the developer.

The practice of developing React or Vue applications is based on developing a Component
that starts with some initial props. The component itself then makes some API calls to
a backend API and load, surprise surprise, state into it's own scope. This state is
then used to render the component into the DOM. Any state change might need to re-render
the component. But the key concept here is that the initial state of the component
is a broken state. It doesn't have the backend data and it will either be "on-loading"
or showing something empty. If data is not loaded, the component needs to decide what
to do about the lack of state.

Inertia spin this into a more manageable approach. Imagine for a second if every PHP
class you instantiated started with some initial values, but you needed to use
Service Locator inside the PHP class to load external state and then make the class
do something with that state. Sounds pretty annoying to me. Inertia makes it so
that your backend dictates which component should be loaded and which property it should
contain. It creates a kind of coupling between the frontend and backend (fullstack app),
but it also eliminates the Vue/React transition through:

- A component exists
- A component is loading data from backend
- A component has loaded data
- A component has rendered

With Inertia the data dependency can live as constructor dependency of your component.
This means that writing unit tests for a component is as hard as mounting it with a 
JSON object hardcoded inside the test itself. No need to figure out how to mock
axios, mock multiple API calls or mount a component while mocking a service locator
data retrieval. It's just a simple JSON object. This is a huge win for me.

#### Vue, Vite, Vitest and Vue-Test-Utils

One thing I didn't expect to learn quickly was that the amount of libraries for React
is a lot bigger than for Vue. Vuetify might seem interesting, but it brings a lot of
"bootstrap-like" development mindset. Frontend developers nowadays prefer to work
with headless components and this area Vue is kind of lacking, to be honest.
Luckily, Tailwind UI and Headless UI cover a big portion of our needs and it's an
amazing product.

Configuring Vite, Vitest and Vue-Test-Utils is pretty easy to figure out. The biggest
hurdle I had was the fact that I don't use a standard Laravek skeleton setup, so 
VITE_URL and ASSET_URL doesn't just work out of the box, I had to figure out how to
properly configure my vite.config.js to use some `base` URL configuration so that I could
build and deploy my frontend into multiple AWS Regions with different subdomains.

It was extremely smart for Vitest to make use of Vite configuration, but add a little `test`
block that just builds up on extra configuration for the project build. This means that
when running my test runner, it is building my application mostly the same way that
production is building it, except that it adds some layers like jsdom.

With Inertia mindset, Vue components are split into 3 categories: Pages, Code Context and Components.
Pages are components that get data from the backend in their constructor and
render it. It's easy to write test for them by having their constructor receiving a JSON
object with the expected data.

Code Context components are just meant to split pages into smaller chunks of code. They
don't necessarily mean reusable components, but rather represent a desire to have less
code crumble in a single file if/when the code allows and make sense. Testing them directly
might not be necessary but even so their data would usually be the same data as the
page or a subset of it.

Components would be your reusable stuff like Button, Modal, Table, Action Context, etc.
These usually involve little domain data and would always get their context from the
parent that are instantiating them anyway, so again testing is made easier.

All these components have an immutable mindset to them, which end up making something less
state-aware and easier to test and predict.


#### Challenges

Not everything is sunshine and rainbows. A few notable challenges I faced coming from
a backend-only development mindset.

package.json cannot be customized like Composer lets us customize "vendor-dir" placement.
This makes it rather harder to customize an application format and "hide away" configuration
code from the business code. The root of the application end up needing to be filled with
things like vite.config.js, tsconfig.json, tailwind.config.js, postcss.config.js, etc.
These files rarely change but creates noise in the project folder structure. We need to
learn to live with it.

vi.mock and vi.doMock. Learn their different as early as you can. I spent a lot of time
trying to figure out why a mock on my 5th test was affecting the entire test file.
Running each test individually would work, but running the entire file would fail.
That's because Javascript has a concept of hoisting. Anything related to vi.mock will
be executed at the top of your file *before* any of your imports are executed.
This makes a lot of sense in hindsight, but when you don't understand what's going on
with javascript code, it's a nightmare to figure out. Basically before your test
imports your modules, vi.mock will prepare a mock for them so that when they are
imported, they have been mocked already. If you hoist your mocks, they affect all
tests in the file. If you opt for vi.doMock, they will not be hoisted and will
allow each test to have different mocks, but that also means you cannot import the
component you're testing at the top of the file, otherwise the underlying thing
that's supposed to be mocked inside the component will just be imported as-is.
A way to bypass this is by doing `await (import ('./MyComponent.vue')).default`
in an async function and calling it only after vi.doMock has been called.

IDE support is still hard. I'm very used to PHPStorm handling everything-php for me.
I don't need to configure anything, it just works. With Javascript, things are a bit
complex. Neither VS Code and PHPStorm are well suited for a great developer experience
yet. Volar is a bit buggy, we need a shim.d.ts to help the IDE understand Vue files and TS
syntax. There's constant error message about Vue module not being found on .vue files
(restarting the Volar server fix those, remember state?). The IDE needs to understand
vite.config.js, tsconfig.json, tailwind.config.js, postcss.config.js, everything! It's
several pieces of individual "single-responsibility" things that we need to figure out
how to stitch together. Nowadays it's a lot easier, but it's still not as easy as
Laravel and PHP itself.

Writing tests for inertia applications in general is not something well documented and
established. Inertia provides helper stuff for Laravel since Laravel already comes with
everything you need to test a Laravel application. Inertia doesn't offer any helper for
Jest or Vitest. It may not be super needed, but it wouldn't hurt to have an axios
mock sample setup so that we can easily write a vitest test for our Inertia Form.submit().
I plan on building up some content on this subject once I'm a bit more comfortable with
the subject.

Typescript has too many configurations. I'm using the default stuff that came with 
something that I installed and I don't even know what it is and so far I'm surviving fine
with it, but obviously 3 months is not enough to dive your head into Inertia, Vue, Vite, 
Vitest, Tailwind and still manage to review all Typescript configurations.

On that note, I haven't had time to figure out eslint either. I'm sure it will be easy, but
I'm just not there yet. This is something I think Laravel Pint made it super easy.

Understanding Teleport and how when something will be rendered in the DOM at the root
of your application and how that makes it hard for you to write tests when mounting
your component is also crucial. It took me several hours to find buried in Vue docs
that if you use .findComponent() instead of .find(), you can find the component that
got teleported.

Writing Laravel/Backend tests with Inertia and mocking Inertia headers is helpful
to test the route.only() behavior as well as testing the 409 response code. Inertia
could have made this easier with their Laravel package, but I actually had to dive
into the HTTP Request my app was making and figure out how to properly reproduce
what Inertia does under the hood.

Declaring two components inside the same .vue file seems like an *impossible* task
if you're using the script setup API. Sometimes it kind of makes sense to give
a name to a piece of code that you want to reference inside your component.
But if that code is small enough, maybe I don't want to extract that to another file.
I would rather avoid having 3 files with 5 lines each and instead have a single file
with 3 component declarations. I'm only able to do this thanks to Jeffrey Way course
on Vue. And still, I'm doing it using the options API because the `defineComponent()`
stuff seems very annoying with Typescript support and PHPStorm intellisense.


#### Conclusion

I'm loving it. I really am. It's the first time in 10 years that I'm trying out a fullstack
application setup and everything makes so much sense and it's quite easy.
The lack of content around this is bothersome, but something I want to tackle myself.
The more I progress with this stack, the more knowledge I gather to start writing about
it. I want to share snippet of codes on how to write tests, configure vite and things
to watch out. This is the first one that I'm writing while still as an infant on this
journey.

Bottom line, Inertia is a genius concept and makes fullstack development a breeze.
Vue could do with some more community components, specially in the headless space,
but it's not a deal breaker. Evan You is definitely rocking it with Vite, Vitest
and Vue. 

I still need to sit down and start writing about CDK as well, but that's a story for
another day.

Share your thoughts with me:

- https://twitter.com/deleugyn
- https://fosstodon.org/@deleugpn
- https://threads.net/@marcodeleu

Cheers.
