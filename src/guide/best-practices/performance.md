---
outline: deep
---
# Performance {#performance}

## Panoramica {#overview}

Vue è progettato per essere performante nella maggior parte dei casi senza bisogno di apportare ottimizzazioni manuali. Tuttavia, ci sono sempre scenari complessi dove è necessaria una messa a punto aggiuntiva. In questa sezione, discuteremo del perché dovresti prestare attenzione quando si parla di performance in un applicazione Vue.

Per prima cosa, discutiamo dei due maggiori aspetti delle performance web:

- Performance del caricamento della pagina: quando veloce la pagina mostra il suo contenuto e diventa interessativa alla visita iniziale. Di solito è misurata usando metriche web vitali come il [Largest Contentful Paint (LCP)](https://web.dev/lcp/) e [First Input Delay (FID)](https://web.dev/fid/).
- Performance d'aggiornamento: quanto velocemente l'applicazione si aggiorna in risposta all'input dell'utente. Per esempio, quando veloce una lista si aggiorna quando un utente scrive nella casella di ricerca, o quanto veloce la pagina passa ad un'altra quando l'utente clicca in un link di navigazione in una Single-Page Application (SPA).

Mentre sarebbe ideale massimizzare entrambe, architetture di frontend diverse tendono a modificare la facilità con cui ci si vuole attenere alle perfomance in questi aspetti. In aggiunta, il tipo di applicazione che stai sviluppando influenza molto gli aspetti prioritari in termini di performance. Pertanto, il primo step da seguire per assicurarsi performance ottimali è selezionare l'architettura corretta per il tipo di applicazione che stai sviluppando:

Consulta i modi di utilizzare Vue per vedere come puoi sfruttare Vue in diversi modi.

- Consulta [Modi di Utilizzare Vue](/guide/extras/ways-of-using-vue) per vedere come puoi sfruttare Vue in diversi modi.
- Jason Miller discute dei tipi di applicazioni web e la rispettiva implementazione ideale in [Application Holotypes](https://jasonformat.com/application-holotypes/).

## Opzioni di Profilazione {#profiling-options}

Per migliorare le performance, dobbiamo prima conoscere come misurarle. Ci sono molteplici strumenti che possono aiutare:

Per profilare le performance di caricamento delle implementazioni di produzione:

- [PageSpeed Insights](https://pagespeed.web.dev/)
- [WebPageTest](https://www.webpagetest.org/)

Per profilare le performance durante lo sviluppo locale:

- [Chrome DevTools Performance Panel](https://developer.chrome.com/docs/devtools/evaluate-performance/)
  - [`app.config.performance`](/api/application#app-config-performance) attiva dei marcatori specifici di Vue nella timeline delle performance dei Chrome DevTools.
- [Vue DevTools Extension](/guide/scaling-up/tooling#browser-devtools) fornisce inoltre una funzionalità per la profilazione.

## Ottimizzazioni caricamento della pagina {#page-load-optimizations}

Ci sono molti aspetti per l'ottimizzazione del caricamento della pagina relativi al framework - dai un'occhiata [a questa guida](https://web.dev/fast/) per un riepilogo completo. Qui, ci concentreremo principalmente a tecniche specifiche di Vue.

### Scegliere l'Architettura Corretta {#choosing-the-right-architecture}

If your use case is sensitive to page load performance, avoid shipping it as a pure client-side SPA. You want your server to be directly sending HTML containing the content the users want to see. Pure client-side rendering suffers from slow time-to-content. This can be mitigated with [Server-Side Rendering (SSR)](/guide/extras/ways-of-using-vue#fullstack-ssr) or [Static Site Generation (SSG)](/guide/extras/ways-of-using-vue#jamstack-ssg). Check out the [SSR Guide](/guide/scaling-up/ssr) to learn about performing SSR with Vue. If your app doesn't have rich interactivity requirements, you can also use a traditional backend server to render the HTML and enhance it with Vue on the client.

If your main application has to be an SPA, but has marketing pages (landing, about, blog), ship them separately! Your marketing pages should ideally be deployed as static HTML with minimal JS, by using SSG.

### Bundle Size and Tree-shaking {#bundle-size-and-tree-shaking}

One of the most effective ways to improve page load performance is shipping smaller JavaScript bundles. Here are a few ways to reduce bundle size when using Vue:

- Use a build step if possible.

  - Many of Vue's APIs are ["tree-shakable"](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking) if bundled via a modern build tool. For example, if you don't use the built-in `<Transition>` component, it won't be included in the final production bundle. Tree-shaking can also remove other unused modules in your source code.
  - When using a build step, templates are pre-compiled so we don't need to ship the Vue compiler to the browser. This saves **14kb** min+gzipped JavaScript and avoids the runtime compilation cost.
- Be cautious of size when introducing new dependencies! In real-world applications, bloated bundles are most often a result of introducing heavy dependencies without realizing it.

  - If using a build step, prefer dependencies that offer ES module formats and are tree-shaking friendly. For example, prefer `lodash-es` over `lodash`.
  - Check a dependency's size and evaluate whether it is worth the functionality it provides. Note if the dependency is tree-shaking friendly, the actual size increase will depend on the APIs you actually import from it. Tools like [bundlejs.com](https://bundlejs.com/) can be used for quick checks, but measuring with your actual build setup will always be the most accurate.
- If you are using Vue primarily for progressive enhancement and prefer to avoid a build step, consider using [petite-vue](https://github.com/vuejs/petite-vue) (only **6kb**) instead.

### Code Splitting {#code-splitting}

Code splitting is where a build tool splits the application bundle into multiple smaller chunks, which can then be loaded on demand or in parallel. With proper code splitting, features required at page load can be downloaded immediately, with additional chunks being lazy loaded only when needed, thus improving performance.

Bundlers like Rollup (which Vite is based upon) or webpack can automatically create split chunks by detecting the ESM dynamic import syntax:

```js
// lazy.js and its dependencies will be split into a separate chunk
// and only loaded when `loadLazy()` is called.
function loadLazy() {
  return import('./lazy.js')
}
```

Lazy loading is best used on features that are not immediately needed after initial page load. In Vue applications, this can be used in combination with Vue's [Async Component](/guide/components/async) feature to create split chunks for component trees:

```js
import { defineAsyncComponent } from 'vue'

// a separate chunk is created for Foo.vue and its dependencies.
// it is only fetched on demand when the async component is
// rendered on the page.
const Foo = defineAsyncComponent(() => import('./Foo.vue'))
```

For applications using Vue Router, it is strongly recommended to use lazy loading for route components. Vue Router has explicit support for lazy loading, separate from `defineAsyncComponent`. See [Lazy Loading Routes](https://router.vuejs.org/guide/advanced/lazy-loading.html) for more details.

## Update Optimizations {#update-optimizations}

### Props Stability {#props-stability}

In Vue, a child component only updates when at least one of its received props has changed. Consider the following example:

```vue-html
<ListItem
  v-for="item in list"
  :id="item.id"
  :active-id="activeId" />
```

Inside the `<ListItem>` component, it uses its `id` and `activeId` props to determine whether it is the currently active item. While this works, the problem is that whenever `activeId` changes, **every** `<ListItem>` in the list has to update!

Ideally, only the items whose active status changed should update. We can achieve that by moving the active status computation into the parent, and make `<ListItem>` directly accept an `active` prop instead:

```vue-html
<ListItem
  v-for="item in list"
  :id="item.id"
  :active="item.id === activeId" />
```

Now, for most components the `active` prop will remain the same when `activeId` changes, so they no longer need to update. In general, the idea is keeping the props passed to child components as stable as possible.

### `v-once` {#v-once}

`v-once` is a built-in directive that can be used to render content that relies on runtime data but never needs to update. The entire sub-tree it is used on will be skipped for all future updates. Consult its [API reference](/api/built-in-directives#v-once) for more details.

### `v-memo` {#v-memo}

`v-memo` is a built-in directive that can be used to conditionally skip the update of large sub-trees or `v-for` lists. Consult its [API reference](/api/built-in-directives#v-memo) for more details.

## General Optimizations {#general-optimizations}

> The following tips affect both page load and update performance.

### Virtualize Large Lists {#virtualize-large-lists}

One of the most common performance issues in all frontend applications is rendering large lists. No matter how performant a framework is, rendering a list with thousands of items **will** be slow due to the sheer number of DOM nodes that the browser needs to handle.

However, we don't necessarily have to render all these nodes upfront. In most cases, the user's screen size can display only a small subset of our large list. We can greatly improve the performance with **list virtualization**, the technique of only rendering the items that are currently in or close to the viewport in a large list.

Implementing list virtualization isn't easy, luckily there are existing community libraries that you can directly use:

- [vue-virtual-scroller](https://github.com/Akryum/vue-virtual-scroller)
- [vue-virtual-scroll-grid](https://github.com/rocwang/vue-virtual-scroll-grid)
- [vueuc/VVirtualList](https://github.com/07akioni/vueuc)

### Reduce Reactivity Overhead for Large Immutable Structures {#reduce-reactivity-overhead-for-large-immutable-structures}

Vue's reactivity system is deep by default. While this makes state management intuitive, it does create a certain level of overhead when the data size is large, because every property access triggers proxy traps that perform dependency tracking. This typically becomes noticeable when dealing with large arrays of deeply nested objects, where a single render needs to access 100,000+ properties, so it should only affect very specific use cases.

Vue does provide an escape hatch to opt-out of deep reactivity by using [`shallowRef()`](/api/reactivity-advanced#shallowref) and [`shallowReactive()`](/api/reactivity-advanced#shallowreactive). Shallow APIs create state that is reactive only at the root level, and exposes all nested objects untouched. This keeps nested property access fast, with the trade-off being that we must now treat all nested objects as immutable, and updates can only be triggered by replacing the root state:

```js
const shallowArray = shallowRef([
  /* big list of deep objects */
])

// this won't trigger updates...
shallowArray.value.push(newObject)
// this does:
shallowArray.value = [...shallowArray.value, newObject]

// this won't trigger updates...
shallowArray.value[0].foo = 1
// this does:
shallowArray.value = [
  {
    ...shallowArray.value[0],
    foo: 1
  },
  ...shallowArray.value.slice(1)
]
```

### Avoid Unnecessary Component Abstractions {#avoid-unnecessary-component-abstractions}

Sometimes we may create [renderless components](/guide/components/slots#renderless-components) or higher-order components (i.e. components that render other components with extra props) for better abstraction or code organization. While there is nothing wrong with this, do keep in mind that component instances are much more expensive than plain DOM nodes, and creating too many of them due to abstraction patterns will incur performance costs.

Note that reducing only a few instances won't have noticeable effect, so don't sweat it if the component is rendered only a few times in the app. The best scenario to consider this optimization is again in large lists. Imagine a list of 100 items where each item component contains many child components. Removing one unnecessary component abstraction here could result in a reduction of hundreds of component instances.
