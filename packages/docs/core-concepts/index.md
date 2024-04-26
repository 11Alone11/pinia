# Определение хранилища %{#defining-a-store}%

<VueSchoolLink
  href="https://vueschool.io/lessons/define-your-first-pinia-store"
  title="Узнайте, как определять и использовать хранилища в Pinia"
/>

Прежде чем погружаться в основные концепции, нужно знать, что хранилище определяется с использованием `defineStore()` и требует уникального названия, передаваемого в качестве первого аргумента:

```js
import { defineStore } from 'pinia'

// Вы можете называть возвращаемое значение defineStore() как угодно,
// но лучше всего использовать имя хранилища и окружить его `use`
// и `Store` (например, `useUserStore`, `useCartStore`, `useProductStore`)
// первый аргумент - это уникальный id хранилища в вашем приложении.
export const useAlertsStore = defineStore('alerts', {
  // остальные параметры...
})
```

Это _название_, также называемое _id_, необходимо и используется Pinia для подключения хранилища к Devtools. Именование возвращаемой функции как _use..._ - это соглашение, которое соблюдается во многих composables, чтобы использование было идиоматичным.

`defineStore()` принимает два различных значения для своего второго аргумента: функцию Setup или объект Options.

## Option-хранилища %{#option-stores}%

Аналогично Options API в Vue, мы также можем передать объект опций со свойствами `state`, `actions` и `getters`.

```js {2-10}
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0, name: 'Иван' }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      this.count++
    },
  },
})
```

Можно представить `state` как `data` хранилища, `getters` как `computed` свойства хранилища и `actions` как `methods`.

Option-хранилища должны быть интуитивными и простыми в использовании для начала работы.

## Setup-хранилища %{#setup-stores}%

Существует также другой возможный синтаксис для определения хранилищ. Аналогично [функции setup](https://vuejs.org/api/composition-api-setup.html) в Composition API Vue, мы можем передать функцию, которая определяет реактивные свойства и методы, и возвращает объект с свойствами и методами, которые мы хотим предоставить.

```js
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const name = ref('Иван')
  const doubleCount = computed(() => count.value * 2)
  function increment() {
    count.value++
  }

  return { count, name, doubleCount, increment }
})
```

В _setup-хранилищах_:

- `ref()` становятся свойствами `состояния`
- `computed()` становятся `геттерами`
- `function()` становятся `действиями`

Обратите внимание, что вы **должны** вернуть **все свойства состояния** в setup-хранилищах, чтобы Pinia могла распознать их как состояние. Другими словами, в хранилищах нельзя иметь _приватные_ свойства состояния. Возвращение не всех свойств состояния может привести к поломке [SSR](../cookbook/composables.md), devtools и других плагинов.

Setup-хранилища предоставляют гораздо большую гибкость по сравнению с [options-хранилищами](#option-stores), так как вы можете создавать наблюдателей (watchers) внутри хранилища и свободно использовать любые [композиции (composables)](https://vuejs.org/guide/reusability/composables.html#composables). Однако имейте в виду, что использование композиций может стать более сложным при использовании рендеринга на стороне сервера (SSR).

Setup-хранилища также могут зависеть от глобальных _предоставленных_ свойств, таких как Router или Route. Любое свойство, [предоставленное на уровне приложения](https://vuejs.org/api/application.html#app-provide), может быть доступно из хранилища с использованием `inject()`, точно так же, как в компонентах:

```ts
import { inject } from 'vue'
import { useRoute } from 'vue-router'
import { defineStore } from 'pinia'

export const useSearchFilters = defineStore('search-filters', () => {
  const route = useRoute()
  // это предполагает, что был вызван `app.provide('appProvided', 'value')`
  const appProvided = inject('appProvided')

  // ...

  return {
    // ...
  }
})
```

:::warning Предупреждение
Не возвращайте свойства, такие как `route `или `appProvided` (из приведенного выше примера), так как они не принадлежат самому хранилищу, и вы можете получить к ним прямой доступ внутри компонентов с помощью `useRoute()` и `inject('appProvided')`.
:::

## Какой синтаксис выбрать? %{#what-syntax-should-i-pick}%

Как и с [Composition API и Options API в Vue](https://vuejs.org/guide/introduction.html#which-to-choose), выбирайте тот подход, с которым вам удобнее всего. У каждого свои преимущества и недостатки. Option-хранилища проще в использовании, в то время как Setup-хранилища более гибкие и мощные. Если вы хотите углубиться в различия, обратитесь к главе [Option Stores vs Setup Stores chapter](https://masteringpinia.com/lessons/when-to-choose-one-syntax-over-the-other) на курсе Mastering Pinia.

## Использование хранилища %{#using-the-store}%

Мы _определяем_ хранилище, потому что хранилище не будет создано, пока не будет вызвано `use...Store()` внутри `<script setup>` компонента (или внутри `setup()` **как и во всех composables**):

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'

// доступ к переменной `store` в любом месте компонента ✨
const store = useCounterStore()
</script>
```

:::tip Совет
Если вы пока не используете компоненты с `setup`, [вы все равно можете использовать Pinia с помощью _map-помощников_](../cookbook/options-api.md).
:::

Вы можете определять столько хранилищ, сколько вам нужно, и **вы должны определять каждое хранилище в разных файлах**, чтобы наилучшим образом использовать возможности Pinia (например, позволить сборщику автоматически разделять код и обеспечивать вывод типов TypeScript).

После создания экземпляра хранилища, вы можете получить доступ к любому свойству, определенному в `state`, `getters` и `actions` напрямую в хранилище. Мы рассмотрим это более подробно на следующих страницах, но автозаполнение поможет вам.

Обратите внимание, что `store` - это объект, обернутый через `reactive`, то есть нет необходимости писать `.value` после геттеров, но, как и `props` в `setup`, **мы не можем его деструктурировать**:

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'
import { computed } from 'vue'

const store = useCounterStore()
// ❌ Это не будет работать, так как нарушает реактивность
// это то же самое, что и деструктуризация из `props`.
const { name, doubleCount } = store // [!code warning]
name // всегда будет "Иван" // [!code warning]
doubleCount // всегда будет 0 // [!code warning]

setTimeout(() => {
  store.increment()
}, 1000)

// ✅ это будет реактивно.
// 💡 но вы также можете просто использовать `store.doubleCount` напрямую
const doubleValue = computed(() => store.doubleCount)
</script>
```

## Деструктуризация из хранилища %{#destructuring-from-a-store}%

Чтобы извлечь свойства из хранилища, сохраняя их реактивность, вам нужно использовать `storeToRefs()`. Он создаст ref-ссылки для каждого реактивного свойства. Это полезно, когда вы используете только состояние из хранилища, но не вызываете никакие действия. Обратите внимание, что вы также можете деструктурировать действия напрямую из хранилища, так как они также привязаны к самому хранилищу:

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'
import { storeToRefs } from 'pinia'

const store = useCounterStore()
// `name` и `doubleCount` являются реактивными ref-ссылками
// При этом также будут извлечены ref-ссылки на свойства, добавленные плагинами
// но будет пропущено любое действие или нереактивное (не ref/reactive) свойство
const { name, doubleCount } = storeToRefs(store)
// действие increment может быть просто деструктурировано
const { increment } = store
</script>
```
