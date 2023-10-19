---
editLink: false
---

[Документация API](../index.md) / [pinia](../modules/pinia.md) / SubscriptionCallbackMutationDirect

# Интерфейс: SubscriptionCallbackMutationDirect

[pinia](../modules/pinia.md).SubscriptionCallbackMutationDirect

Контекст, который передается в коллбек подписки при прямом изменении состояния хранилища с помощью `store.someState = newValue` или `store.$state.someState = newValue`.

## Иерархия

- [`_SubscriptionCallbackMutationBase`](pinia._SubscriptionCallbackMutationBase.md)

  ↳ **`SubscriptionCallbackMutationDirect`**

## Свойства

### events

• **events**: `DebuggerEvent`

🔴 ТОЛЬКО ДЛЯ РАЗРАБОТКИ, НЕ использовать в production коде. Различные вызовы изменений. Поступают от https://vuejs.org/guide/extras/reactivity-in-depth.html#reactivity-debugging и позволяют отслеживать изменения в devtools и плагинах **только во время разработки**.

#### Переопределяет

[_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[events](pinia._SubscriptionCallbackMutationBase.md#events)

___

### storeId

• **storeId**: `string`

`id` хранилища, осуществляющего изменение.

#### Наследуется от

[_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[storeId](pinia._SubscriptionCallbackMutationBase.md#storeId)

___

### type

• **type**: [`direct`](../enums/pinia.MutationType.md#direct)

Тип изменения

#### Переопределяет

[_SubscriptionCallbackMutationBase](pinia._SubscriptionCallbackMutationBase.md).[type](pinia._SubscriptionCallbackMutationBase.md#type)
