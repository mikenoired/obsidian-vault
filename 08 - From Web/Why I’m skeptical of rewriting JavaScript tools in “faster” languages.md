---
День: 2025-09-09
url: https://nolanlawson.com/2024/10/20/why-im-skeptical-of-rewriting-javascript-tools-in-faster-languages/
tags:
  - "#Web"
---
I’ve written a lot of JavaScript. I _like_ JavaScript. And more importantly, I’ve built up a set of skills in understanding, optimizing, and debugging JavaScript that I’m reluctant to give up on.

So maybe it’s natural that I get a worried pit in my stomach over the current mania to rewrite every Node.js tool in a “faster” language like Rust, Zig, Go, etc. Don’t get me wrong – these languages are cool! (I’ve got a copy of the Rust book on my desk right now, and I even contributed [a bit](https://github.com/servo/servo/commits?author=nolanlawson) to Servo for fun.) But ultimately, I’ve invested a ton of my career in learning the ins and outs of JavaScript, and it’s by far the language I’m most comfortable with.

So I acknowledge my bias (and perhaps over-investment in one skill set). But the more I think about it, the more I feel that my skepticism is also justified by some real objective concerns, which I’d like to cover in this post.

## Performance

One reason for my skepticism is that I just don’t think we’ve exhausted all the possibilities of making JavaScript tools faster. Marvin Hagemeister has done [an excellent job](https://marvinh.dev/blog/speeding-up-javascript-ecosystem/) of demonstrating this, by showing how much low-hanging fruit there is in ESLint, Tailwind, etc.

In the browser world, JavaScript has proven itself to be “fast enough” for most workloads. Sure, WebAssembly exists, but I think it’s fair to say that it’s mostly used for niche, CPU-intensive tasks rather than for building a whole website. So why are JavaScript-based CLI tools rushing to throw JavaScript away?
### The big rewrite

I think the perf gap comes from a few different things. First, there’s the aforementioned low-hanging fruit – for a long time, the JavaScript tooling ecosystem has been focused on building something that _works_, not something fast. Now we’ve reached a saturation point where the API surface is mostly settled, and everyone just wants “the same thing, but faster.” Hence the explosion of new tools that are nearly drop-in replacements for existing ones: [Rolldown](https://rolldown.rs/) for [Rollup](https://rollupjs.org/), [Oxlint](https://oxc.rs/docs/guide/usage/linter) for [ESLint](https://eslint.org/), [Biome](https://biomejs.dev/formatter/) for [Prettier](https://prettier.io/), etc.

However, these tools aren’t necessarily faster because they’re using a faster language. They could just be faster because 1) they’re being written with performance in mind, and 2) the API surface is already settled, so the authors don’t have to spend development time tinkering with the overall design. Heck, you don’t even need to write tests! Just use the existing test suite from the previous tool.

In my career, I’ve often seen a rewrite from A to B resulting in a speed boost, followed by the triumphant claim that B is faster than A. However, [as Ryan Carniato points out](https://www.youtube.com/live/0F9t_WeJ5p4?t=4234s), a rewrite is often faster just because it’s a rewrite – you know more the second time around, you’re paying more attention to perf, etc.
### Bytecode and JIT

The second class of performance gaps comes from the things browsers give us for free, and that we rarely think about: the [bytecode cache](https://v8.dev/blog/code-caching-for-devs) and [JIT](https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/) (Just-In-Time compiler).

When you load a website for the second or third time, if the JavaScript is cached correctly, then the browser doesn’t need to parse and compile the source code into bytecode anymore. It just loads the bytecode directly off disk. This is the bytecode cache in action.

Furthermore, if a function is “hot” (frequently executed), it will be further optimized into machine code. This is the JIT in action.

In the world of Node.js scripts, we don’t get the benefits of the bytecode cache at all. Every time you run a Node script, the entire script has to be parsed and compiled from scratch. This is a big reason for the reported perf wins between JavaScript and non-JavaScript tooling.

Thanks to the inimitable [Joyee Cheung](https://joyeecheung.github.io/blog/about/), though, Node is now getting a [compile cache](https://github.com/nodejs/node/pull/52535). You can set an environment variable and immediately get faster Node.js script loads:

```bash
export NODE_COMPILE_CACHE=~/.cache/nodejs-compile-cache
```

I’ve set this in my `~/.bashrc` on all my dev machines. I hope it makes it into the default Node settings someday.

As for JIT, this is another thing that (sadly) most Node scripts can’t really benefit from. You have to run a function before it becomes “hot,” so on the server side, it’s more likely to kick in for long-running servers than for one-off scripts.

And the JIT can make a big difference! In Pinafore, I considered replacing the JavaScript-based [blurhash](https://blurha.sh/) library with a Rust (Wasm) version, before [realizing](https://github.com/nolanlawson/pinafore/pull/1781#issuecomment-630562314) that the performance difference was erased by the time we got to the fifth iteration. That’s the power of the JIT.

Maybe eventually a tool like [Porffor](https://github.com/CanadaHonk/porffor) could be used to do an AOT (Ahead-Of-Time) compilation of Node scripts. In the meantime, though, JIT is still a case where native languages have an edge on JavaScript.

I should also acknowledge: there is a perf hit from using Wasm versus pure-native tools. So this could be another reason native tools are taking the CLI world by storm, but not necessarily the browser frontend.

## Contributions and debuggability

I hinted at it earlier, but this is the main source of my skepticism toward the “rewrite it all in native” movement.

JavaScript is, in my opinion, a working-class language. It’s very forgiving of types (this is one reason I’m not a huge TypeScript fan), it’s easy to pick up (compared to something like Rust), and since it’s supported by browsers, there is a huge pool of people who are conversant with it.

For years, we’ve had both library authors and library consumers in the JavaScript ecosystem largely using JavaScript. I think we take for granted what this enables.

For one: the path to contribution is much smoother. To quote [Matteo Collina](https://fosstodon.org/@mcollina/112723450963851116):

> [!quote]
> Most developers ignore the fact that they have the skills to debug/fix/modify their dependencies. They are not maintained by unknown demigods but by fellow developers.

This breaks down if JavaScript library authors are using languages that are different (and more difficult!) than JavaScript. They may as well be demigods!

For another thing: it’s straightforward to modify JavaScript dependencies locally. I’ve often tweaked something in my local `node_modules` folder when I’m trying to track down a bug or work on a feature in a library I depend on. Whereas if it’s written in a native language, I’d need to check out the source code and compile it myself – a big barrier to entry.

(To be fair, this has already gotten a bit tricky thanks to the widespread use of TypeScript. But TypeScript is not _too_ far from the source JavaScript, so you’d be amazed how far you can get by clicking “pretty print” in the DevTools. Thankfully most Node libraries are also not minified.)

Of course, this also leads us back to debuggability. If I want to debug a JavaScript library, I can simply use the browser’s DevTools or a Node.js debugger that I’m already familiar with. I can set breakpoints, inspect variables, and reason about the code as I would for my own code. This [isn’t impossible with Wasm](https://developer.chrome.com/blog/wasm-debugging-2020), but it requires a different skill set.

## Conclusion

I think it’s great that there’s a new generation of tooling for the JavaScript ecosystem. I’m excited to see where projects like [Oxc](https://oxc.rs/) and [VoidZero](https://voidzero.dev/posts/announcing-voidzero-inc) end up. The existing incumbents are indeed exceedingly slow and would probably benefit from the competition. (I get especially peeved by the typical `eslint` + `prettier` + `tsc` + `rollup` lint+build cycle.)

That said, I don’t think that JavaScript is inherently slow, or that we’ve exhausted all the possibilities for improving it. Sometimes I look at truly perf-focused JavaScript, such as the [recent improvements](https://learn.microsoft.com/en-us/microsoft-edge/devtools-guide-chromium/whats-new/2024/08/devtools-128#heap-snapshot-improvements) to the Chromium DevTools using mind-blowing techniques like [using `Uint8Array`s as bit vectors](https://github.com/ChromeDevTools/devtools-frontend/commit/b73fc5a44552e81019b614594ba7c375f74fc446), and I feel that we’ve barely scratched the surface. (If you really want an inferiority complex, see [other commits from Seth Brenith](https://github.com/ChromeDevTools/devtools-frontend/commits?author=sethbrenith). They are wild.)

I also think that, as a community, we have not really grappled with what the world would look like if we relegate JavaScript tooling to an elite priesthood of Rust and Zig developers. I can imagine the average JavaScript developer feeling completely hopeless every time there’s a bug in one of their build tools. Rather than empowering the next generation of web developers to achieve more, we might be training them for a career of learned helplessness. Imagine what it will feel like for the average junior developer to face a [segfault](https://en.wikipedia.org/wiki/Segmentation_fault) rather than a familiar JavaScript `Error`.

At this point, I’m a senior in my career, so of course I have little excuse to cling to my JavaScript security-blanket. It’s part of my job to dig down a few layers deeper and understand how every part of the stack works.

However, I can’t help but feel like we are embarking down an unknown path with unintended consequences, when there is another path that is less fraught and could get us nearly the same results. The current freight train shows no signs of slowing down, though, so I guess we’ll find out when we get there.
