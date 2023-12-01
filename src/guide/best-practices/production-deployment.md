# Distribuzione in Production {#production-deployment}

## Sviluppo vs. Produzione {#development-vs-production}

Durante lo sviluppo, Vue fornisce delle funzionalità per migliorare l'esperienza di sviluppo:

- Avvertimenti per errori e insidie comuni
- Validazione di parametri / eventi
- [Hooks per il debugging della reattività](/guide/extras/reactivity-in-depth#reactivity-debugging)
- Integrazioni dei devtools

Tuttavia, queste funzionalità diventano inutili in produzione. Alcuni controlli degli avvertimenti possono inoltre causare un piccolo calo di prestazioni. Quando si distribuisce in production, dobbiamo rimuovere tutti i branch inutili o di sviluppo per avere un carico di dimensioni minori e performance maggiori.

## Senza build tools {#without-build-tools}

Se stai usando Vue senza un build tool caricandolo da un CDN o da uno script self-hosted, assicurati di usare la build di production (i file dist che finiscono con `.prod.js`) quando distribuisci in production. Le build production sono pre-minificate con tutti i branch di solo sviluppo rimossi.

- Se stai usando una buld globale (accessibile tramite `Vue` globale): usa `vue.global.prod.js`.
- Se stai usando una build ESM (accessibile tramite l'importazione nativa di ESM): usa `vue.esm-browser.prod.js`.

Consulta la guida al [dist file](https://github.com/vuejs/core/tree/main/packages/vue#which-dist-file-to-use) per maggiori informazioni.

## Con build tools {#with-build-tools}

I progetti generati tramite `create-vue` (basato su Vite) o Vue CLI (basato su webpack) sono pre-configurati per le build di produzione.


Se stai usando un setup personalizzato, assicurati che:

1. `vue` si risolve in `vue.runtime.esm-bundler.js`.
2. The [compile time feature flags](https://github.com/vuejs/core/tree/main/packages/vue#bundler-build-feature-flags) are properly configured.
3. I [flag della funzione di compilazione di tempo](https://github.com/vuejs/core/tree/main/packages/vue#bundler-build-feature-flags) siano configurati correttamente.
4. <code>process.env<wbr>.NODE_ENV</code> sia rimpiazzato con `"production"` durante la build.

Riferimenti aggiuntivi:

- [Vite production build guide](https://vitejs.dev/guide/build.html)
- [Vite deployment guide](https://vitejs.dev/guide/static-deploy.html)
- [Vue CLI deployment guide](https://cli.vuejs.org/guide/deployment.html)

## Tracciare errori di esecuzione {#tracking-runtime-errors}

l'[app-level error handler](/api/application#app-config-errorhandler) può essere usato per inviare errori ai servizi di tracking:

```js
import { createApp } from 'vue'

const app = createApp(...)

app.config.errorHandler = (err, instance, info) => {
  // report error to tracking services
}
```

Servizi come [Sentry](https://docs.sentry.io/platforms/javascript/guides/vue/) e [Bugsnag](https://docs.bugsnag.com/platforms/javascript/vue/) offrono anche integrazioni ufficiali per Vue.
