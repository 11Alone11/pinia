# Миграция с 0.x (v1) до v2 %{#migrating-from-0-x-v1-to-v2}%

С версии `2.0.0-rc.4` Pinia поддерживает как Vue 2, так и Vue 3! Это означает, что все новые обновления будут применяться к версии 2, чтобы пользователи Vue 2 и Vue 3 могли извлечь из этого выгоду. Если вы используете Vue 3, это ничего не меняет для вас, так как вы уже использовали версию rc, и вы можете проверить [список изменений](https://github.com/vuejs/pinia/blob/v2/packages/pinia/CHANGELOG.md) для подробного объяснения всех изменений. В противном случае, **это руководство для вас**!

## Устаревшее %{#deprecations}%

Давайте рассмотрим все изменения, которые вам нужно внести в свой код. Прежде всего, убедитесь, что вы уже используете последнюю версию 0.x, чтобы увидеть какие-либо устаревшие функции:

```shell
npm i 'pinia@^0.x.x'
# или с помощью yarn
yarn add 'pinia@^0.x.x'
```

Если вы используете ESLint, рассмотрите возможность использования [этого плагина](https://github.com/gund/eslint-plugin-deprecation) для поиска всех устаревших использований. В противном случае вы должны сразу видеть их как перечеркнутые. Вот API, которые были устаревшими и удалены:

- `createStore()` становится `defineStore()`
- В подписках `storeName` становится `storeId`
- `PiniaPlugin` был переименован в `PiniaVuePlugin` (плагин Pinia для Vue 2)
- `$subscribe()` больше не принимает _boolean_ в качестве второго параметра, вместо него передавайте объект с `detached: true`.
- Плагины Pinia больше не получают напрямую `id` хранилища. Вместо этого используйте `store.$id`.

## Несовместимые обновления %{#breaking-changes}%

После их удаления можно перейти на версию 2 с помощью:

```shell
npm i 'pinia@^2.x.x'
# или с помощью yarn
yarn add 'pinia@^2.x.x'
```

И начните обновлять свой код.

### Общий тип хранилища %{#generic-store-type}%

Добавлен в [2.0.0-rc.0](https://github.com/vuejs/pinia/blob/v2/packages/pinia/CHANGELOG.md#200-rc0-2021-07-28)

Замените любое использование типа `GenericStore` на `StoreGeneric`. Это новый общий тип хранилища, который должен принимать любой тип хранилища. Если вы писали функции, использующие тип `Store` без передачи его дженериков (например, `Store<Id, State, Getters, Actions>`), то вам также следует использовать `StoreGeneric`, так как тип `Store` без дженериков создает пустой тип хранилища.

```ts
function takeAnyStore(store: Store) {} // [!code --]
function takeAnyStore(store: StoreGeneric) {} // [!code ++]

function takeAnyStore(store: GenericStore) {} // [!code --]
function takeAnyStore(store: StoreGeneric) {} // [!code ++]
```

## `DefineStoreOptions` для плагинов %{#definestoreoptions-for-plugins}%

Если вы писали плагины, используя TypeScript, и расширяли тип `DefineStoreOptions` для добавления пользовательских опций, то вам следует переименовать его в `DefineStoreOptionsBase`. Этот тип будет применяться как к setup-хранилищам, так и к option-хранилищам.

```ts
declare module 'pinia' {
  export interface DefineStoreOptions<S, Store> { // [!code --]
  export interface DefineStoreOptionsBase<S, Store> { // [!code ++]
    debounce?: {
      [k in keyof StoreActions<Store>]?: number
    }
  }
}
```

## `PiniaStorePlugin` был переименован %{#piniastoreplugin-was-renamed}%

Тип `PiniaStorePlugin` был переименован в `PiniaPlugin`.

```ts
import { PiniaStorePlugin } from 'pinia' // [!code --]
import { PiniaPlugin } from 'pinia' // [!code ++]

const piniaPlugin: PiniaStorePlugin = () => { // [!code --]
const piniaPlugin: PiniaPlugin = () => { // [!code ++]
  // ...
}
```

**Примечание: данное изменение возможно только после обновления до последней версии Pinia без устаревших использований**.

## Версия `@vue/composition-api` %{#-vue-composition-api-version}%

Поскольку pinia теперь полагается на `effectScope()`, необходимо использовать как минимум версию `1.1.0` пакета `@vue/composition-api`:

```shell
npm i @vue/composition-api@latest
# или с помощью yarn
yarn add @vue/composition-api@latest
```

## Поддержка webpack 4 %{#webpack-4-support}%

Если вы используете webpack 4 (Vue CLI использует webpack 4), вы можете столкнуться с ошибкой следующего вида:

```
ERROR  Failed to compile with 18 errors

 error  in ./node_modules/pinia/dist/pinia.mjs

Can't import the named export 'computed' from non EcmaScript module (only default export is available)
```

Это связано с модернизацией файлов dist для поддержки нативных ESM-модулей в Node.js. Теперь файлы используют расширение `.mjs` и `.cjs`, тобы позволить Node воспользоваться этим преимуществом. Чтобы исправить эту проблему, у вас есть два варианта:

- Если вы используете Vue CLI 4.x, обновите свои зависимости. Это должно включать исправление, приведенное ниже.
  - Если обновление для вас невозможно, добавьте это в ваш `vue.config.js`:

    ```js
    // vue.config.js
    module.exports = {
      configureWebpack: {
        module: {
          rules: [
            {
              test: /\.mjs$/,
              include: /node_modules/,
              type: 'javascript/auto',
            },
          ],
        },
      },
    }
    ```

- Если вы вручную управляете webpack, то вам придется указать ему, как работать с файлами `.mjs`:

  ```js
  // webpack.config.js
  module.exports = {
    module: {
      rules: [
        {
          test: /\.mjs$/,
          include: /node_modules/,
          type: 'javascript/auto',
        },
      ],
    },
  }
  ```

## Devtools %{#devtools}%

Pinia v2 больше не перехватывает Vue Devtools v5, для этого требуется Vue Devtools v6. Найдите ссылку на скачивание на странице [Документация Vue Devtools](https://devtools.vuejs.org/guide/installation.html#chrome) для **бета-канала** расширения.

## Nuxt %{#nuxt}%

Если вы используете Nuxt, у Pinia теперь есть собственный пакет для Nuxt 🎉. Установите его с помощью:

```bash
npm i @pinia/nuxt
# или с помощью yarn
yarn add @pinia/nuxt
```

Также не забудьте **обновить пакет `@nuxtjs/composition-api`**.

Затем адаптируйте свои `nuxt.config.js` и `tsconfig.json`, если вы используете TypeScript:

```js
// nuxt.config.js
module.exports {
  buildModules: [
    '@nuxtjs/composition-api/module',
    'pinia/nuxt', // [!code --]
    '@pinia/nuxt', // [!code ++]
  ],
}
```

```json
// tsconfig.json
{
  "types": [
    // ...
    "pinia/nuxt/types" // [!code --]
    "@pinia/nuxt" // [!code ++]
  ]
}
```

Рекомендуется также ознакомиться с [специальным разделом Nuxt](../ssr/nuxt.md).
