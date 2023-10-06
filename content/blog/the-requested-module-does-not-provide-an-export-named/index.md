---
title: The requested module does not provide an export named
date: "2023-10-05T21:10:32.017Z"
description: Annoying and hard to debug issue with Vue and Inertia
---

I've been working on a project using [Inertia](https://inertiajs.com/) and
[Vue](https://vuejs.org/). One of the things I was doing was to share
a property from Laravel to Vue. I made a costly mistake of naming
my variable uppercase on Laravel.

```php
return Inertia::render('Users', [
    'MyProperty' => 'MyPropertyValue'
]);
```

In order to work with this property, I defined an interface on Typescript and
called the `defineProps` method on my Vue component

```ts
export interface User {
    MyProperty: string;
}
```

```vue
import {User} from '/my/path/to/user';

<script setup lang="ts">
const props = defineProps<User>();
</script>
```

This property totally works fine. It gets super annoying when you try to pass this property
down to a underlying component. I was doing something like this:

```vue
<template>
    <div>
        <MyComponent :some-prop="props.MyProperty" />
    </div>
</template>
```

As soon as you do this, you get the following error:

> Uncaught (in promise) SyntaxError: The requested module '...' 
> does not provide an export named 'User' (at ....)

What is super annoying about this is that it crashes at the interface definition.
No matter how much you dig around, you keep looking at the inevitable fact that
1) the interface exists 
2) it is being exported 
3) the path is correctly imported.

It makes you doubt your sanity a little bit, which is why I decided
to write this post. This edge-case comes from the fact that folks
working with Vue will know better not to name their properties in
a way that is incompatible with HTML and Javascript simultaneously
as it is documented here: https://vuejs.org/guide/components/props.html#prop-passing-details.
The problem becomes more challenging when we throw Inertia into
the mix because sharing data from Laravel to Vue with Inertia will
work fine with a PascalCase property name. It is only when you try
to define this on a Typescript interface and pass it down to a
child component that it breaks in such an unexpected way.

Hope you enjoyed the reading. If you're interested in more of my crazy
ideas, follow me on [Twitter](https://twitter.com/deleugyn).

Cheers.