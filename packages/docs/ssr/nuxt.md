# Nuxt.js %{#nuxt-js}%

Использование Pinia с [Nuxt](https://nuxt.com/) проще, так как Nuxt заботится о многих аспектах _рендеринга на стороне сервера_. Например, **вам не нужно беспокоиться о сериализации и XSS-атаках**. Pinia поддерживает Nuxt Bridge и Nuxt 3. Для поддержки чистого Nuxt 2, [см. ниже](#nuxt-2-without-bridge).

## Установка %{#installation}%

```bash
yarn add pinia @pinia/nuxt
# или с помощью npm
npm install pinia @pinia/nuxt
```

:::tip Совет
Если вы используете npm, возможно, вы столкнетесь с ошибкой _ERESOLVE unable to resolve dependency tree_. В этом случае добавьте следующее в ваш `package.json`:

```js
"overrides": {
  "vue": "latest"
}
```

:::

Мы предоставляем _модуль_, который обрабатывает все для вас, вам нужно только добавить его в раздел `modules` в файле `nuxt.config.js`:

```js
// nuxt.config.js
export default defineNuxtConfig({
  // ... другие параметры
  modules: [
    // ...
    '@pinia/nuxt',
  ],
})
```

И вот, используйте свое хранилище как обычно!

## Ожидание выполнения действий на страницах %{#awaiting-for-actions-in-pages}%

Как и с `onServerPrefetch()`, вы можете вызывать действие хранилища внутри `asyncData()`. Учитывая, как работает `useASyncData()`, убедитесь, что возвращаете значение. Это позволит Nuxt пропустить выполнение действия на стороне клиента и повторно использовать значение с сервера.

```vue{3-4}
<script setup>
const store = useStore()
// мы также могли бы извлечь данные, но они уже присутствуют в хранилище
await useAsyncData('user', () => store.fetchUser())
</script>
```

Если ваше действие не возвращает значение, вы можете добавить любое значение, не являющееся `null` или `undefined`:

```vue{3}
<script setup>
const store = useStore()
await useAsyncData('user', () => store.fetchUser().then(() => true))
</script>
```

:::tip Совет

Если вы хотите использовать хранилище за пределами `setup()`, не забудьте передать объект `pinia` в `useStore()`. Мы добавили его в [контекст](https://nuxtjs.org/docs/2.x/internals-glossary/context), поэтому у вас есть к нему доступ в `asyncData()` и `fetch()`:

```js
import { useStore } from '~/stores/myStore'

export default {
  asyncData({ $pinia }) {
    const store = useStore($pinia)
  },
}
```

:::

## Автоматические импорты %{#auto-imports}%

По умолчанию `@pinia/nuxt` предоставляет несколько автоматических импортов:

- `usePinia()`, похоже на `getActivePinia()`, но лучше работает с Nuxt. Вы можете добавить автоматические импорты, чтобы упростить свою жизнь:
- `defineStore()` для определения хранилищ
- `storeToRefs()`когда необходимо извлечь отдельные ref-ссылки из хранилища
- `acceptHMRUpdate()` для [горячей замены модулей](../cookbook/hot-module-replacement.md)

Также автоматически импортируются **все хранилища**, определенные в вашей папке `stores`. Однако вложенные хранилища не ищутся. Вы можете настроить это поведение, установив параметр `storesDirs`:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  // ... другие параметры
  modules: ['@pinia/nuxt'],
  pinia: {
    storesDirs: ['./stores/**', './custom-folder/stores/**'],
  },
})
```

Обратите внимание, что пути папок относительны к корню вашего проекта. Если вы измените параметр `srcDir`, вам также придется изменить пути папок.

## Nuxt 2 без bridge %{#nuxt-2-without-bridge}%

Pinia поддерживает Nuxt 2 до версии `@pinia/nuxt` v0.2.1. Также убедитесь, что вместе с pinia вы установили [`@nuxtjs/composition-api`](https://composition-api.nuxtjs.org/) наряду с pinia.

```bash
yarn add pinia @pinia/nuxt@0.2.1 @nuxtjs/composition-api
# или при помощи
npm install pinia @pinia/nuxt@0.2.1 @nuxtjs/composition-api
```

Мы поставляем _модуль_, который будет выполнять все действия за вас, вам нужно только добавить его в `buildModules` в файле `nuxt.config.js`:

```js
// nuxt.config.js
export default {
  // ... другие параметры
  buildModules: [
    // Только для Nuxt 2:
    // https://composition-api.nuxtjs.org/getting-started/setup#quick-start
    '@nuxtjs/composition-api/module',
    '@pinia/nuxt',
  ],
}
```

### TypeScript %{#typescript}%

Если вы используете Nuxt 2 (`@pinia/nuxt` < 0.3.0) с TypeScript или у вас есть `jsconfig.json`, вам также следует добавить типы для `context.pinia`:

```json
{
  "types": [
    // ...
    "@pinia/nuxt"
  ]
}
```

Это также обеспечит автозаполнение 😉 .

### Использование Pinia совместно с Vuex %{#using-pinia-alongside-vuex}%

Рекомендуется **избегать совместного использования Pinia и Vuex**, но если необходимо использовать и то, и другое, то нужно указать pinia, чтобы она не отключала Vuex:

```js
// nuxt.config.js
export default {
  buildModules: [
    '@nuxtjs/composition-api/module',
    ['@pinia/nuxt', { disableVuex: false }],
  ],
  // ... другие параметры
}
```
