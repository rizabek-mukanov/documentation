[refs-public-api]: /docs/concepts/public-api.md

# Viewer

> Работа с авторизованным пользователем согласно методологии

---

Во всех приложениях так или иначе есть бизнес-логика, завязанная **на текущем авторизованном пользователе.**

> Обычно такая сущность называется `Viewer` / `Principle` / `Session` - но в рамках статьи, остановимся именно на `viewer`, но все зависит от вашего проекта и команды

Также, это один из показательных примеров, когда бизнес-сущность порождает за собой бизнес-фичи, затем страницы, и даже бизнес-процессы

Рассмотрим их подробнее ниже с примерами

> **Примечание 1:** Названия директорий внутри сегментов (ui, model) могут отличаться от проекта к проекту
>
> *Методология пока никак не регламентирует этот уровень вложенности*

> **Примечание 2:** Стоит также понимать, что приведенные ниже примеры - абстрактны и синтетичны, и сформированы точкой зрения и опытом автора
>
> *Реализация в ваших проектах может отличаться, методология, опять же, никак не регламентирует эти аспекты*

## Entities

> - *`Can use:` shared*
> - *`Used by:` features, pages, processes, app*

**Бизнес-сущность пользователя**

- Представляет собой наиболее атомарную абстракцию для проектирования
- Здесь формируется контекст авторизации, на который потом обычно полагаются все вышележащие слои приложения

> Стоит понимать, что нередко в приложении есть публичный "внешний" пользователь (`user`), а есть авторизованный "внутренний" пользователь (`viewer`)
>
> *Не забывайте учитывать эту разницу при проектировании архитектуры и логики*

### Примеры

```sh
# Сегменты могут быть как файлами, так и директориями
|
├── entities/viewer              # Layer: Бизнес-сущности
|         |                      #     Slice: Текущий пользователь
|         ├── ui/                #      Segment: UI-логика (компоненты)
|         ├── lib/               #      Segment: Инфраструктурная-логика (хелперы)
|         ├── model/             #      Segment: Бизнес-логика
|         └── index.ts           #      [Декларация Public API]
|   ...           
```

- `entities/viewer` - сущность текущего пользователя *(Session / Principle)*
- `entities/user` - сущность публичного пользователя *(не обязательно связанное с текущим)*
  - *В зависимости от сложности приложения - можно использовать и `user` для нейминга текущего пользователя*
  - *Но это может вызвать серьезные проблемы, когда/если придется разделять логику обычного пользователя и текущего, который зашел в систему*

### `index.ts`

Обычный [Public API модуля][refs-public-api]

*Во многом повторяет декларацию API и для описанных ниже слоев*

```ts
// ui.ts
export { ViewerCard } from "./card";
export { ViewerAvatar } from "./avatar";
...
```

```ts
// model.ts
export * as selectors from "./selectors";
export * as events from "./events";
export * as stores from "./stores";
...
```

```ts
// index.ts
export * from "./ui"
export * as viewerModel from "./model";
```

### `ui`

Здесь могут содержаться компоненты, относящиеся не к конкретной странице/фиче, а напрямую к сущности пользователя

```tsx
import { Card } from "shared/ui/card";

// Считается хорошей практикой - не связывать напрямую с моделью ui-компоненты из entitites
// Чтобы можно было использовать не только для текущей модели,
// Но и для поступивших извне пропсов

export type UserCardProps = {
    data: User;
    className?: string;
    // И прочие card-пропсы
};

export const UserCard = ({ data, ... }: UserCardProps) => {
    return (
        <Card 
            title={data.fullName}
            description={data.bio}
            ...
        />
    )
}
```

### `model`

На этом уровне обычно создается сущность текущего пользователя, с реэкспортом хуков/контрактов/селекторов для использования вышележащими слоями

```ts
// effector
export const $user = createStore(...);
// redux (+ toolkit)
export const userSlice = createSlice(...)
```

```ts
// effector
export const useViewer = () => {
    return useStore($user)
}
// redux (+ toolkit)
export const useViewer = () => {
    return useSelector((store) => store.entities.userSlice);
}
```

Также тут может быть реализована и другая логика

- `updateUserDetails`
- `logoutUser`
- ...

## Features

> - *`Can use:` shared, entities*
> - *`Used by:` pages, processes, app*

**Фичи, завязанные на текущем пользователе**

- Использует в реализации бизнес-сущности (зачастую - `entities/viewer`) и shared ресурсы
- Фичи могут не быть напрямую связаны с вьювером, но при этом могут использовать его контекст при реализации логики

### Примеры

```sh
# Сегменты могут быть как файлами, так и директориями
|
├── features/auth                # Layer: Бизнес-фичи
|        |                       #    Slice Group: Структурная группа "Авторизация пользователя"
|        ├── by-phone/           #      Slice: Фича "Авторизация по телефону"
|        |     ├── ui/           #         Segment: UI-логика (компоненты)
|        |     ├── lib/          #         Segment: Инфраструктурная-логика (хелперы)
|        |     ├── model/        #         Segment: Бизнес-логика
|        |     └── index.ts      #         [Декларация Public API]
|        |
|        ├── by-oauth/           #      Slice: Фича "Авторизация по внешнему ресурсу"
|   ...           
```

- `features/auth/{by-phone, by-oauth, logout ...}` - **структурная** группа фич авторизации *(по телефону, по внешнему ресурсу, выход из системы, ...)*
- `features/wallet/{add-funds, ...}` - **структурная** группа фич по работе со внутренним счетом пользователя *(пополнение счета, ...)*

### `ui`

- Авторизация по внешнему ресурсу

```tsx
import { viewerModel } from "entities/viewer";

export const AuthByOAuth = () => {
    return (
        <OAuth
            domain={...}
            scope={...}
            ...
            // для redux - дополнительно нужен dispatch
            onSuccess=((user) => viewerModel.setUser(user))
        />
    )
}
```

- Использование контекста пользователя в фичах

```tsx
import { viewerModel } from "entities/viewer";

export const Wallet = () => {
    const viewer = viewerModel.useViewer();
    const { moneyCount } = wallet;
    
    ...
}
```

- Использование компонентов вьювера

```tsx
import { ViewerAvatar } from "entities/viewer";
...
export const Header = () => {
    ...
    return (
        <Layout.Header>
            ...
            <ViewerAvatar
                onClick={...}
                onLogout={...}
                ...
            />
        </Layout.Header>
    )
}
```

## Pages

> - *`Can use:` shared, entities, features*
> - *`Used by:` processes, app*

**Страницы, так или иначе связанные с текущим пользователем**

- Могут как напрямую затрагивать функциональность вьювера
- Так и использовать его косвенно (в том числе - и его контекст / фичи)

### Примеры

```sh
# Сегменты могут быть как файлами, так и директориями
|
├── pages/viewer                 # Layer: Страницы приложения
|        |                       #    Slice Group: Структурная группа "Текущий пользователь"
|        ├── profile/            #     Slice: Страница профиля вьювера
|        |     ├── ui.tsx        #         Segment: UI-логика (компоненты)
|        |     ├── lib.ts        #         Segment: Инфраструктурная-логика (хелперы)
|        |     ├── model.ts      #         Segment: Бизнес-логика
|        |     └── index.ts      #         [Декларация Public API]
|        |
|        ├── settings/           #     Slice: Страница настроек аккаунта вьювера
|   ...           
```

- `pages/viewer/profile` - страница ЛК пользователя
- `pages/viewer/settings` - страница настроек аккаунта пользователя
- `pages/user` - страница пользователя (не обязательно текущего)
- `pages/auth/{sign-in, sign-up, reset}` - **структурная** группа страниц авторизации *(вход в систему / регистрация / восстановление пароля)*

### `ui`

- Использование компонентов вьювера и *viewer-based* фич на страницах

```tsx
import { Wallet } from "features/wallet";
import { ViewerCard } from "entities/viewer";
...
export const UserPage = () => {
    ...
    return (
        <Layout>
            <Header
                extra={<Wallet.AddFunds />}
            />
            ...
            <ViewerCard />
        </Layout>
    )
}
```

- Использование модели вьювера

```tsx
import { viewerModel } from "entities/viewer";
...
export const SomePage = () => {
    ...
    return (
        <Layout>
            ...
            <Settings onSave={(payload) => viewerModel.events.saveChanges(payload)} />
        </Layout>
    )
}
```

## Processes

> - *`Can use:` shared, entities, features, pages*
> - *`Used by:` app*

**Бизнес-процессы, затрагивающие текущего пользователя**

- Затрагивает юзкейсы, пронизывающие страницы системы
- **Слой процессов - опционален**, и обычно используется *только когда логика разрастается в страницах* и нужно *отдельное управление контекстом* на сразу нескольких страницах

### Примеры

```sh
# Сегменты могут быть как файлами, так и директориями
|
├── processes                    # Layer: Бизнес процессы
|        ├── auth/               #     Slice: Процесс авторизации пользователя
|        |     ├── lib.ts        #         Segment: Инфраструктурная-логика (хелперы)
|        |     ├── model.ts      #         Segment: Бизнес-логика
|        |     └── index.ts      #         [Декларация Public API]
|        |
|        ├── quick-tour/         #     Slice: Процесс онбординга нового пользователя
|   ...           
```

- `processes/auth` - бизнес-процесс авторизации пользователя
- `processes/quick-tour` - бизнес-процесс для ознакомления пользователя с системой *(~ UserOnboard)*

## App

> - *`Can use:` shared, entities, features, pages, processes*
> - *`Used by:` -*

**Инициализация данных по учетной записи пользователя**

- Как правило, здесь проходит проверка на то, **был ли уже авторизован пользователь** до того, как зашел в сервис
  - **Если да** - провайдер пополняет контекст пользователя, для дальнейшего использования в системе
  - **Если нет** - запускается процесс авторизации или меняется контекст вьювера, чтобы страница авторизации предприняла нужные действия

### Примеры

```sh
# Структура `app` уникальна для каждого проекта и не регламентируется методологией
|
├── app/hocs                     # Layer: Инициализация приложения (HOCs провайдеры)
|        ├── withAuth.tsx        #    HOC: Инициализация контекста авторизации
|        |   ...                 #
|   ...           
```

- `app/hocs/withAuth` - HOC для авторизации пользователя
  - Используется **только на верхнем уровне, как провайдер** с инициализацией логики, к которому имеет доступ только *`app`-слой*
  - **Не путать с хуком `useViewer`**, к которому идет обращение всех остальных слоев *(processes / pages / features)*

## Выводы

Как мы видим на примерах выше - **вся бизнес-логика начинает строится от одной сущности, и может распространится до самого верхнего слоя.**

Поэтому нужно уметь **грамотно выделять скоуп влияния модуля**, и в зависимости от этого выбирать подходящий слой для расположения логики.

*Таким образом - мы получим наибоолее поддерживаемый, читаемый и переиспользуемый код.*

## См. также

- [Дискуссия "Применимость feature-sliced в бою"](https://github.com/feature-sliced/wiki/discussions/65) (*внутри также есть примеры с viewer*)
- [Вопрос про профиль и сущность пользователя (community-chat)](https://t.me/feature_sliced/342)
- [Пояснение про UIKIT#Card и User#Card (community-chat)](https://t.me/feature_sliced/552)