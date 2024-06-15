+++
title = 'Mrandom'
date = 2024-06-13T18:01:23-04:00
draft = false

description = 'Fixing a node package I made a while back.'
type = "post"
showTableOfContents = true
tags = ["node", "typescript", "npm", "tech"]
+++
## A Pythonic Random Module for JavaScripters and TypeScripters
A while back (8 months ago apparently...) I got fed up with the built-in `Math.random()` function in JavaScript/in the Browser. 
It's not that it's *bad* per se, but it's not very flexible and pretty opaque. For example, it's not even possible to set the seed for the random number generator.
Oh, and if you want an integer between 0 and n, you have to do something like `Math.floor(Math.random() * n)`. If you want to get a random element from an array, you have to do something like `arr[Math.floor(Math.random() * arr.length)]`. 
In the year of our Lord 2024, I think we can do better.

All of that got me thinking, "Dang, I wish I had Python's `random` module in JavaScript." So, I made it. Sure, there are probably a million other modules that do this or something similar
(or something even better), but I wanted to make my own. So I did. And it's called `mrandom`. It's on npm, you can check it out [here](https://www.npmjs.com/package/@mnovo323/mrandom).

I really haven't looked at too many other libraries/modules/packages, but at a glance I actually think mine has some merit:
- Fully typed (TypeScript)
- 1-to-1 with Python's `random` module (okay I still need to implement vonmisesvariate, paretovariate, weibullvariate, but cut me some slack)
- Seedable
- Uses xoshiro128+ (a pretty good PRNG that I ported myself to TypeScript from some C looking pseudocode)

Is it cryptographically secure? No, probably not. But it's good enough for most use cases. I think. I hope.

## Where I Last Left Off
Last I worked on this, I got the idea on the clock, got off work, started coding, and called it done at like 11:00 PM. I was pleased with myself about it, too. I wrote a couple of tests for it, built it out, published it to npm, and called it a day.

Over the next few days I got pretty excited seeing that there were hundreds of people (bots, most likely) downloading my package! Then I thought, "Oh, gosh. I don't think I even tried to import my package and test it in a separate project." So, I did that. And it didn't work. Whoops. The code is fine, I just messed up something with the building and publishing, methinks.

I get easily distracted by other projects, so I didn't get back to it until now. I'm going to fix it up, add some more tests, and maybe even add some more functionality. Let's see how it goes.

## Trying To Read Old Code
I cloned the [repo](https://www.github.com/mnovo323/mrandom), ran the tests, and I've got 5/68 failing. Honestly, I'm not convinced it's the code, I could have written bad tests. ![Image showing 3 failed tests](/images/mrandom/mrandom_failed_tests.png "Failing Tests") Not great, but before I fix that, I want to see if I can even get the package to work in a separate project. I'm going to start by creating a new project and importing `mrandom` into it. Let's see how it goes.

```bash
❯ npm install @mnovo323/mrandom

added 1 package, and audited 2 packages in 2s

found 0 vulnerabilities
```

Noice. Let's take a look in the package to see what got distributed.

```bash
❯ ls
dist  jest.config.js  package.json  README.md  src  tests  tsconfig.json
```

Ah, yes. Only the essentials. The README, source, tests, and dist. Brilliantly done, definitely didn't only need the dist directory.

Well, In any case, let's try to import it and use it.

```typescript
import { randint } from '@mnovo323/mrandom';

const n = randint(1, 10);

console.log(n);
```

After compiling it with tsc and trying to run it, we get:
```bash
❯ node dist/index.js
node:internal/modules/cjs/loader:1186
  const err = new Error(message);
              ^

Error: Cannot find module '../src/randint'
Require stack:
- /home/michaelnovotny/Documents/test_npm/node_modules/@mnovo323/mrandom/dist/choice.js
- /home/michaelnovotny/Documents/test_npm/node_modules/@mnovo323/mrandom/dist/index.js
    at Module._resolveFilename (node:internal/modules/cjs/loader:1186:15)
    at Module._load (node:internal/modules/cjs/loader:1012:27)
    at Module.require (node:internal/modules/cjs/loader:1271:19)
    at require (node:internal/modules/helpers:123:16)
    at Object.<anonymous> (/home/michaelnovotny/Documents/test_npm/node_modules/@mnovo323/mrandom/dist/choice.js:4:19)
    at Module._compile (node:internal/modules/cjs/loader:1434:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1518:10)
    at Module.load (node:internal/modules/cjs/loader:1249:32)
    at Module._load (node:internal/modules/cjs/loader:1065:12)
    at Module.require (node:internal/modules/cjs/loader:1271:19) {
  code: 'MODULE_NOT_FOUND',
  requireStack: [
    '/home/michaelnovotny/Documents/test_npm/node_modules/@mnovo323/mrandom/dist/choice.js',
    '/home/michaelnovotny/Documents/test_npm/node_modules/@mnovo323/mrandom/dist/index.js'
  ]
}

Node.js v22.2.0
```

Yikes.

Before tackling that, I checked my failing test suite, and lo and behold I wrote a bug in my code. Quick fix, but thanks testing!

After a lot of back and forth with GPT-4o, reading numerous blogs, and trial and error, I finally found out what my main problem is: I don't know how node *really* works. It is embarrassing. I use node packages all the time
and I look through them from time to time to see how they were implemented, but I guess you can only learn my doing. Which brings me to...

## A Million Ways To Skin A Cat
There's too many tools and too many ways to do things in the JavaScript ecosystem. I'm not even talking about the frameworks and libraries, I'm talking about the build tools, the testing tools, the linting tools, the formatting tools, the CI/CD tools, the package managers, the bundlers, the transpiler, the... You get the point. 

Laugh at me if you want, but if I'm being honest, I've never made a node project from scratch. It's always `create-X-app` which gives me some boiler plate to ruin. I've never even
had to understand the difference between CommonJS and ES Modules (aside from sometimes you have to use `import` and sometimes you have to use `require`). When I started tinkering with my tsconfig, I got very confused by the sheer amount of options for 'module' (I went with ESNext, by the way).
There's too many ways to do things to do things. I'm not even sure what best practices are for this sort of thing. My requirements are something like:
- Fully typed, with the ability to use the package in both JavaScript and TypeScript
- Import like `import { randint, choice } from '@mnovo323/mrandom'`
- Import like `import { randint } from '@mnovo323/mrandom/randint'`
- Import like `import random from '@mnovo323/mrandom'`
  - then use like `random.randint(1, 10)`
- Import like `const random = require('@mnovo323/mrandom')`
- Import like `const randint = require('@mnovo323/mrandom/randint')`

The specific imports are important for tree shaking, I don't want to force a dev to import the entire module if they only need one function.

After googling a whole lot and hitting my head against a brick wall, currently I have a `dist` directory with the following structure:
```bash
dist
├── index.d.mts
├── index.d.ts
├── index.js
└── index.mjs
```

And my source directory has the following structure:
```bash
src
├── betavariate.ts
├── binomialvariate.ts
├── choices.ts
├── choice.ts
├── expovariate.ts
├── gammavariate.ts
├── gauss.ts
├── getrandbits.ts
├── index.ts
├── lognormvariate.ts
├── randbytes.ts
├── randint.ts
├── randrange.ts
├── sample.ts
├── shuffle.ts
├── triangular.ts
├── uniform.ts
└── xoshiro128plus.ts
```

I didn't like that there was a single index, and also there is no need for a .d.mts and a .d.ts. After some work I got it to look like this:
```bash
dist
├── betavariate.js
├── betavariate.mjs
├── binomialvariate.js
├── binomialvariate.mjs
├── choice.js
├── choice.mjs
├── choices.js
├── choices.mjs
├── expovariate.js
├── expovariate.mjs
├── gammavariate.js
├── gammavariate.mjs
├── gauss.js
├── gauss.mjs
├── getrandbits.js
├── getrandbits.mjs
├── index.js
├── index.mjs
├── lognormvariate.js
├── lognormvariate.mjs
├── randbytes.js
├── randbytes.mjs
├── randint.js
├── randint.mjs
├── randrange.js
├── randrange.mjs
├── sample.js
├── sample.mjs
├── shuffle.js
├── shuffle.mjs
├── triangular.js
├── triangular.mjs
├── types
│   ├── betavariate.d.ts
│   ├── binomialvariate.d.ts
│   ├── choice.d.ts
│   ├── choices.d.ts
│   ├── expovariate.d.ts
│   ├── gammavariate.d.ts
│   ├── gauss.d.ts
│   ├── getrandbits.d.ts
│   ├── index.d.ts
│   ├── lognormvariate.d.ts
│   ├── randbytes.d.ts
│   ├── randint.d.ts
│   ├── randrange.d.ts
│   ├── sample.d.ts
│   ├── shuffle.d.ts
│   ├── triangular.d.ts
│   ├── uniform.d.ts
│   └── xoshiro128plus.d.ts
├── uniform.js
├── uniform.mjs
├── xoshiro128plus.js
└── xoshiro128plus.mjs
```

I used tsup to create .mjs and .js files for everything including the index, I had my tsconfig only output types to a ./dist/types directory, and I had my package.json point to the index and also every import individually. You can see my configs and dist structure [here](https://www.github.com/mnovo323/mrandom).

## Concluding Thoughts
Working in JavaScript/TypeScript land can be a nightmare, but I learned a lot through this experience about how the whole ecosystem works right now, and in the process put out a pretty cool package.

I'm more than positive I have a lot more to learn in this space, and I would really like to have a more experienced dev look through my project and give me some pointers. I'm sure there are a million things I could do better.
