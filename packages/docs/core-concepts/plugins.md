# Плагины %{#plugins}%

Хранилища Pinia можно полностью расширить благодаря низкоуровневому API. Вот список вещей, которые вы можете сделать:

- Добавление новых свойств в хранилища
- Добавление новых опций при определении хранилищ
- Добавление новых методов в хранилища
- Оборачивание существующих методов
- Перехват действий и их результатов
- Реализация таких побочных эффектов (side-effects), как [Local Storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)
- Применять **только** к определенным хранилищам

Плагины добавляются в экземпляр pinia с помощью `pinia.use()`. Простейший пример - добавление статического свойства ко всем хранилищам, путев возврата объекта:

```js
import { createPinia } from 'pinia'

// добавить свойство `secret` в каждое создаваемое хранилище
// после установки плагина это свойство может находиться в другом файле
function SecretPiniaPlugin() {
  return { secret: 'the cake is a lie' }
}

const pinia = createPinia()
// передать плагин pinia
pinia.use(SecretPiniaPlugin)

// в другом файле
const store = useStore()
store.secret // 'the cake is a lie'
```

Это полезно для добавления глобальных объектов, таких как роутер, модальные окна или менеджеры уведомлений.

## Вступление %{#introduction}%

Плагин Pinia - это функция, которая по желанию возвращает свойства, добавляемые в хранилище. Она принимает один необязательный аргумент - _контекст_:

```js
export function myPiniaPlugin(context) {
  context.pinia // pinia, созданное с помощью `createPinia()`
  context.app // текущее приложение, созданное с помощью `createApp()` (только для Vue 3)
  context.store // хранилище, которое дополняется плагином
  context.options // объект опций, переданный в `defineStore()`, определяющий хранилище
  // ...
}
```

Затем эта функция передается в `pinia` с помощью функции `pinia.use()`:

```js
pinia.use(myPiniaPlugin)
```

Плагины применяются только к хранилищам, созданным **после самих плагинов и после передачи `pinia` в приложение**, иначе они не будут применены.

## Расширение хранилища %{#augmenting-a-store}%

Вы можете добавлять свойства в каждое хранилище, просто возвращая их объект в плагине:

```js
pinia.use(() => ({ hello: 'world' }))
```

Вы также можете установить свойство напрямую в `store`, но **по возможности используйте версию с возвратом, чтобы они могли быть автоматически отслеживаемыми с помощью devtools**:

```js
pinia.use(({ store }) => {
  store.hello = 'world'
})
```

Любое свойство, _возвращаемое_ плагином, будет автоматически отслеживаться devtools. Чтобы сделать `hello` видимым в devtools, убедитесь, что добавили его в `store._customProperties` **только в режиме разработки**, если вы хотите отлаживать его в devtools:

```js
// из примера выше
pinia.use(({ store }) => {
  store.hello = 'world'
  // убедитесь, что ваш сборщик обработает этот код. webpack и vite должны делать это по умолчанию
  if (process.env.NODE_ENV === 'development') {
    // добавьте любые ключи, которые вы установили в хранилище
    store._customProperties.add('hello')
  }
})
```

Обратите внимание, что каждое хранилище оборачивается в [`reactive`](https://vuejs.org/api/reactivity-core#reactive), автоматически раскрывая любой Ref (`ref()`, ` computed()`, ...), который оно содержит:

```js
const sharedRef = ref('shared')
pinia.use(({ store }) => {
  // у каждого хранилища есть свое собственное свойство `hello`
  store.hello = ref('secret')
  // он автоматически разворачивается
  store.hello // 'secret'

  // все хранилища совместно используют свойство `shared`
  store.shared = sharedRef
  store.shared // 'shared'
})
```

Именно поэтому ко всем вычисляемым свойствам можно обращаться без `.value` и именно поэтому они являются реактивными.

### Добавление нового состояния %{#adding-new-state}%

Если вы хотите добавить в хранилище новые свойства состояния или свойства, которые предназначены для использования во время гидратации, **вам придется добавить их в двух местах**:

- В `store`, чтобы вы могли получить к нему доступ с помощью `store.myState`
- В `store.$state`, чтобы его можно было использовать в devtools и **сериализовать во время SSR**.

Кроме того, для разделения значения при разных обращениях к нему, конечно же, придется использовать `ref()` (или другой реактивный API):

```js
import { toRef, ref } from 'vue'

pinia.use(({ store }) => {
  // чтобы правильно обработать SSR, нам нужно убедиться, что мы не
  // переопределяем существующее значение
  if (!store.$state.hasOwnProperty('hasError')) {
    // hasError определяется внутри плагина, поэтому каждое хранилище имеет свое
    // индивидуальное свойство состояния
    const hasError = ref(false)
    // установка переменной на `$state` позволяет сериализовать ее во время SSR
    store.$state.hasError = hasError
  }
  // нам нужно перенести ref-ссылку из состояния в хранилище, таким образом
  // оба доступа: store.hasError и store.$state.hasError будут работать
  // и совместо использовать одну и ту же переменную
  // См. https://vuejs.org/api/reactivity-utilities.html#toref
  store.hasError = toRef(store.$state, 'hasError')

  // в этом случае лучше не возвращать hasError, так как он
  // все равно будет отображаться в разделе state в devtools
  // и если мы его вернем, devtools отобразят его дважды.
})
```

Обратите внимание, что изменения состояние или его дополнения, которые происходят внутри плагина (включая вызов `store.$patch()`), происходят до того, как хранилище будет активным и, следовательно, **не вызывают никаких подписок**.

:::warning Предупреждение
Если вы используете**Vue 2**, Pinia подвержена [тем же ограничениям реактивности](https://v2.vuejs.org/v2/guide/reactivity.html#Change-Detection-Caveats), что и Vue. При создании новых свойств состояния, таких как `secret` и `hasError`, вам нужно будет использовать `Vue.set()` (для Vue 2.7) или `set()` (из `@vue/composition-api` для Vue <2.7):

```js
import { set, toRef } from '@vue/composition-api'
pinia.use(({ store }) => {
  if (!store.$state.hasOwnProperty('secret')) {
    const secretRef = ref('secret')
    // Если данные предназначены для использования во время SSR, их следует
    // устанавливать в свойстве $state, чтобы они сериализовались и
    // подхватывались во время гидратации
    set(store.$state, 'secret', secretRef)
  }
  // установить его непосредственно в хранилище, чтобы вы могли получить к нему доступ
  // обоими способами: `store.$state.secret` / `store.secret`
  set(store, 'secret', toRef(store.$state, 'secret'))
  store.secret // 'secret'
})
```

:::

#### Сброс состояния, добавленного в плагинах %{#resetting-state-added-in-plugins}%

По умолчанию `$reset()` не сбрасывает состояние, добавленное плагинами, но вы можете переопределить его, чтобы сбрасывалось и состояние, которое вы добавляете:

```js
import { toRef, ref } from 'vue'

pinia.use(({ store }) => {
  // для справки, это тот же код, что и выше
  if (!store.$state.hasOwnProperty('hasError')) {
    const hasError = ref(false)
    store.$state.hasError = hasError
  }
  store.hasError = toRef(store.$state, 'hasError')

  // обязательно установите контекст (`this`) на хранилище
  const originalReset = store.$reset.bind(store)

  // переопределение функции $reset
  return {
    $reset() {
      originalReset()
      store.hasError = false
    },
  }
})
```

## Добавление новых внешних свойств %{#adding-new-external-properties}%

При добавлении внешних свойств, экземпляров классов, полученных из других библиотек, или просто вещей, не являющихся реактивными, перед передачей объекта в pinia следует обернуть его с помощью `markRaw()`. Приведем пример добавления роутера в каждое хранилище:

```js
import { markRaw } from 'vue'
// адаптируйте это в зависимости от того, где находится ваш маршрутизатор
import { router } from './router'

pinia.use(({ store }) => {
  store.router = markRaw(router)
})
```

## Вызов `$subscribe` внутри плагинов %{#сalling-subscribe-inside-plugins}%

Вы можете использовать [store.$subscribe](./state.md#subscribing-to-the-state) и [store.$onAction](./actions.md#subscribing-to-actions) и внутри плагинов:

```ts
pinia.use(({ store }) => {
  store.$subscribe(() => {
    // реагировать на изменения в хранилище
  })
  store.$onAction(() => {
    // реагировать на дейстия в хранилище
  })
})
```

## Добавление новых опций %{#adding-new-options}%

При определении хранилищ можно создавать новые опции, чтобы впоследствии использовать их в плагинах. Например, можно создать опцию `debounce`, которая позволяет отменить любое действие:

```js
defineStore('search', {
  actions: {
    searchContacts() {
      // ...
    },
  },

  // в дальнейшем это будет прочитано плагином
  debounce: {
    // задержать выполнение действия searchContacts на 300мс
    searchContacts: 300,
  },
})
```

Затем плагин может прочитать эту опцию для оборачивания действий и замены исходных:

```js
// используйте любую библиотеку debounce
import debounce from 'lodash/debounce'

pinia.use(({ options, store }) => {
  if (options.debounce) {
    // мы переопределяем действия на новые
    return Object.keys(options.debounce).reduce((debouncedActions, action) => {
      debouncedActions[action] = debounce(
        store[action],
        options.debounce[action]
      )
      return debouncedActions
    }, {})
  }
})
```

Обратите внимание, что при использовании setup-синтаксиса пользовательские опции передаются в качестве 3-го аргумента:

```js
defineStore(
  'search',
  () => {
    // ...
  },
  {
    // в дальнейшем это будет прочитано плагином
    debounce: {
      // задержать выполнение действия searchContacts на 300мс
      searchContacts: 300,
    },
  }
)
```

## TypeScript %{#typescript}%

Все, что показано выше, может быть сделано с поддержкой типизации, так что вам никогда не понадобится использовать `any` или `@ts-ignore`.

### Типизация плагинов %{#typing-plugins}%

Плагин Pinia может быть типизирован следующим образом:

```ts
import { PiniaPluginContext } from 'pinia'

export function myPiniaPlugin(context: PiniaPluginContext) {
  // ...
}
```

### Типизация новых свойств хранилища %{#typing-new-store-properties}%

При добавлении новых свойств в хранилища, необходимо также расширять интерфейс `PiniaCustomProperties`.

```ts
import 'pinia'
import type { Router } from 'vue-router'

declare module 'pinia' {
  export interface PiniaCustomProperties {
    // используя сеттер, мы можем разрешить использование как строк, так и ref-ссылок
    set hello(value: string | Ref<string>)
    get hello(): string

    // можно определять и более простые значения
    simpleNumber: number

    // типизация роутера, добавленного плагином выше (#adding-new-external-properties)
    router: Router
  }
}
```

После этого его можно безопасно писать и читать:

```ts
pinia.use(({ store }) => {
  store.hello = 'Hola'
  store.hello = ref('Hola')

  store.simpleNumber = Math.random()
  // @ts-expect-error: we haven't typed this correctly
  store.simpleNumber = ref(Math.random())
})
```

`PiniaCustomProperties` - это общий тип, позволяющий ссылаться на свойства хранилища. Представьте себе следующий пример, в котором мы копируем исходные опции как `$options` (это будет работать только для option-хранилищ):

```ts
pinia.use(({ options }) => ({ $options: options }))
```

Мы можем правильно типизировать его, используя 4 дженерика `PiniaCustomProperties`:

```ts
import 'pinia'

declare module 'pinia' {
  export interface PiniaCustomProperties<Id, S, G, A> {
    $options: {
      id: Id
      state?: () => S
      getters?: G
      actions?: A
    }
  }
}
```

:::tip Совет
При расширении типов в дженериках они должны быть названы **точно так же, как в исходном коде**. `Id` не может быть назван `id` или `I`, а `S` не может быть назван `State`. Вот что означает каждая буква:

- S: Состояние
- G: Геттеры
- A: Действия
- SS: Setup-хранилище / хранилище

:::

### Типизация нового состояния %{#typing-new-state}%

При добавлении новых свойств состояния (как в `store`, так и в `store.$state`), вы должны добавить тип в `PiniaCustomStateProperties` вместо этого. В отличие от `PiniaCustomProperties`, в него передается только дженерик `State`:

```ts
import 'pinia'

declare module 'pinia' {
  export interface PiniaCustomStateProperties<S> {
    hello: string
  }
}
```

### Типизация новых опций создания %{#typing-new-creation-options}%

При создании новых опций для `defineStore()`, вы должны расширять `DefineStoreOptionsBase`. В отличие от `PiniaCustomProperties`, в него передаются только два дженерика: State и Store, позволяя вам ограничить то, что можно определить. Например, вы можете использовать названия действий:

```ts
import 'pinia'

declare module 'pinia' {
  export interface DefineStoreOptionsBase<S, Store> {
    // позволяет определить тип number для мс для любого из действий
    debounce?: Partial<Record<keyof StoreActions<Store>, number>>
  }
}
```

:::tip Совет
Существует также тип `StoreGetters` для извлечения _геттеров_ из типа Store. Вы также можете расширить опции _setup-хранилищ_ или _option-хранилищ_ **только**, расширив типы `DefineStoreOptions` и `DefineSetupStoreOptions` соответственно.
:::

## Nuxt.js %{#nuxt-js}%

При [использовании pinia вместе с Nuxt](../ssr/nuxt.md) необходимо сначала создать [Nuxt плагин](https://nuxt.com/docs/guide/directory-structure/plugins). Это даст вам доступ к экземпляру `pinia`:

```ts{14-16}
// plugins/myPiniaPlugin.ts
import { PiniaPluginContext } from 'pinia'

function MyPiniaPlugin({ store }: PiniaPluginContext) {
  store.$subscribe((mutation) => {
    // реагировать на изменения  хранилища
    console.log(`[🍍 ${mutation.storeId}]: ${mutation.type}.`)
  })

  // Обратите внимание, что это должно быть типизировано, если вы используете TS
  return { creationTime: new Date() }
}

export default defineNuxtPlugin(({ $pinia }) => {
  $pinia.use(MyPiniaPlugin)
})
```

::: info Для справки

В приведенном примере используется TypeScript, поэтому при использовании файла `.js` необходимо удалить аннотации типов `PiniaPluginContext` и `Plugin`, а также их импорт.

:::

### Nuxt.js 2 %{#nuxt-js-2}%

Если вы используете Nuxt.js 2, то типы немного отличаются:

```ts{3,15-17}
// plugins/myPiniaPlugin.ts
import { PiniaPluginContext } from 'pinia'
import { Plugin } from '@nuxt/types'

function MyPiniaPlugin({ store }: PiniaPluginContext) {
  store.$subscribe((mutation) => {
    // реагировать на изменения  хранилища
    console.log(`[🍍 ${mutation.storeId}]: ${mutation.type}.`)
  })

  // Обратите внимание, что это должно быть типизировано, если вы используете TS
  return { creationTime: new Date() }
}

const myPlugin: Plugin = ({ $pinia }) => {
  $pinia.use(MyPiniaPlugin)
}

export default myPlugin
```
