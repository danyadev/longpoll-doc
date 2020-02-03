# User Long Poll API

### План документации:
1. [Подключение](#подключение)
2. [Возвращаемые ошибки](#возвращаемые-ошибки)
3. [Получение устаревшей истории событий](#получение-устаревшей-истории-событий)
4. [Структура сообщения](#структура-сообщения)
5. [Описание событий](#описание-событий)
   - [Событие 2. Установка флагов сообщения](#событие-2-установка-флагов-сообщения)
6. [Дополнительные данные](#дополнительные-данные)
   - [Флаги сообщений](#флаги-сообщений)
   - [Сервисные сообщения](#сервисные-сообщения)
   - [Клавиатура для ботов](#клавиатура-для-ботов)

Документация написана для __11__ версии Long Poll.

## Подключение

__Long Polling__ - это способ получения событий в реальном времени используя бесконечную цепочку запрос - ответ - запрос... При подаче запроса сервер возвращает ответ не сразу, а когда придет новое событие или истечет время ожидания.

Ссылку для запроса нужно генерировать следующим образом:

> https://**`server`**?act=a_check&key=**`key`**&ts=**`ts`**&wait=**`wait`**&mode=**`mode`**&version=**`version`**

* `server`, `key` и `ts` получаются один раз методом [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer)
* `version` - Версия Long Poll
* `wait` - Время ожидания нового события в секундах, максимум 90
* `mode` - Дополнительные опции ответа:
  * `2` - Возвращать вложения
  * `8` - Возвращать нормальные данные некоторых событий
  * `32` - Возвращать `pts`
  * `64` - Возвращать данные о платформе в событии онлайна друга
  * `128` - Возвращать `random_id`

В `Node.js` ссылку можно составить следующим образом:
```js
// server, key и ts нужно получить заранее. 
const link = `https://${server}?` + require('querystring').stringify({
  act: 'a_check',
  key: key,
  ts: ts,
  wait: 10,
  mode: 2 | 8 | 32 | 64 | 128,
  version: 11
});
```

После выполнения запроса сервер вернет ответ следующего вида:
```js
{
  updates?: Array,
  ts?: Number,
  pts?: Number,
  failed?: Number,
  min_version?: Number,
  max_version?: Number
}
```

Затем нужно будет обработать пришедшие в `updates` события и повторить запрос, перед этим заменив `ts` на новый из ответа.

## Возвращаемые ошибки

Иногда вместо поля `updates` в ответе может прийти поле `failed`. В первой половине случаев эта ошибка очень легко решается и не связана с потерей событий (2 и 4), но во второй, чтобы восстановить все "потерянные" события, нужно будет немножко попотеть.

1. Устарела история событий. Решается [получением и обработкой устаревшей истории событий](#получение-устаревшей-истории-событий) и использованием переданного `ts` далее.
2. Истекло время действия ключа. Решается использованием `key` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
3. Информация о пользователе утрачена. Решается [получением и обработкой устаревшей истории событий](#получение-устаревшей-истории-событий) и использованием `key` и `ts` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
4. Передана неправильная версия. Вместе с ошибкой передаются поля `min_version` и `max_version`, так что проблем здесь не возникнет.

## Получение устаревшей истории событий

Чтобы получить устаревшую историю событий, нам необходим `pts`, получение которого включается полем `need_pts: 1` в [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer) и добавлением в `mode` флага `32` при [выполнении запроса](#подключение).

Для получении истории мы будем использовать метод [`messages.getLongPollHistory`](https://vk.com/dev/messages.getLongPollHistory) с указанием следующих параметров при запросе:
- `ts` - последний полученный `ts` из лонгпула
- `pts` - последний полученный `pts` из лонгпула
- `msgs_limit` - минимум `200`, рекомендую `500`
- `max_msg_id` - `id` последнего полученного сообщения из лонгпула
- `onlines` - `1` если возвращать события `8` и `9` (онлайн и оффлайн друга), `0` если нет
- `lp_version` - последняя версия лонгпула
- `fields` - поля [пользователей и групп](https://vk.com/dev/objects/user), которые придут вместе с историей.

Ответ выглядит следующим образом:
```js
{
  history: Array,
  messages: { count: Number, items: Array },
  conversations: Array,
  from_pts: Number,
  new_pts: Number,
  profiles?: Array,
  groups?: Array,
  more?: Number
}
```

Поле `history` идентично полю `updates` в Long Poll, за исключением сокращения некоторых событий:
- `3` - сброс флага сообщения
- `4` - новое сообщение
- `5` - редактирование сообщения
- `18` - добавление сниппета к сообщению

В результате сокращения события будут выглядеть так:
```
[event_id, msg_id, flags, peer_id]
```

Основную информацию о сообщениях и беседах нужно будет брать из полей `messages` и `conversations`.

Рекомендую __НЕ__ обрабатывать `18` событие, потому что в `4` событии сниппет уже будет прикреплен к сообщению. 

Если в ответе придет поле `more`, то после обработки всех событий нужно будет повторить запрос, указав в поле `pts` пришедший `new_pts`, а в поле `max_msg_id` `id` последнего полученного сообщения отсюда.

## Структура сообщения

Самой сложной структурой события в LongPoll является структура сообщения, которая идентична сразу для нескольких событий. В ней есть еще 4 довольно сложных момента, которые помечены как `<Type>` и описаны в отдельных абзацах:

- [Flags](#флаги-сообщений) - Флаги сообщений
- [Keyboard](#клавиатура-для-ботов) - Клавиатура для ботов (в том числе инлайн)
- [Action](#сервисные-сообщения) - Сервисное сообщение
- Attachments - Вложения

```js
[
  event_id: 3 || 4 || 5 || 18,
  msg_id: Number,
  flags<Flags>: Number,
  peer_id: Number,
  timestamp: Number,
  text: String,
  {
    title?: ' ... ', // Приходит только в лс
    emoji?: '1', // Наличие emoji (кто бы мог подумать...)
    from?: String, // id автора сообщения. Приходит только в беседах
    fwd_count?: Number, // В последнее время не приходит..
    has_template?: '1', // Наличие карусели
    mentions?: Array, // Массив с id людей, которых пушнули или ответили на их сообщения
    marked_users?: Array, // Плохой аналог mentions, игнорируем
    keyboard<Keyboard>?: Object, // Клавиатура для ботов (в т.ч. инлайн)
    <Action> // Описание сервисного сообщения
  },
  {
    fwd?: '0_0', // При пересланных сообщениях и ответах на сообщения
    <Attachments> // Описание вложений
  },
  random_id: Number,
  conversation_msg_id: Number,
  edit_time: Number // 0 (не отредактированно) или timestamp (время редактирования)
]
```

## Описание событий

### Событие 2. Установка флагов сообщения
Структура: `[2, msg_id, flags, peer_id]`

Это событие вызывается в 5 случаях, которые отличаются [флагами сообщения](#флаги-сообщений):
1. Пометка важным (`important`)
2. Пометка как спам (`spam`)
3. Удаление сообщения (`deleted`)
4. Удаление для всех (`deleted` и `deleted_all`)
5. Прослушка голосового сообщения (`audio_listened`)

## Дополнительные данные

### Флаги сообщений

Маска представляет собой сумму флагов, которые являются степенью двойки.
Обычно маску записывают таким образом:
```js
// Так принято: 
const mask = 1 | 2 | 8 | 64 | 1024 | 1 << 11 | 1 << 15;
// Но можно и так:
// Скобки необходимы, потому что у "+" приоритет выше чем у "<<"
const mask = 1 + 2 + 8 + 64 + 1024 + (1 << 11) + (1 << 15);
```

Так как при увеличении степени двойки числа тоже увеличиваются, и довольно быстро,
можно использовать `1 << n` или `2 ** n`, что означает `2` в степени `n`.

|    Название    |                 Описание                 |  Значение  |
| :------------: | :--------------------------------------: | :--------: |
| unread         | Непрочитанное сообщение                  | `1 << 0`   |
| outbox         | Исходящее сообщение                      | `1 << 1`   |
| important      | Важное сообщение                         | `1 << 3`   |
| chat           | Отправка сообщения в беседу через vk.com | `1 << 4`   |
| friends        | Исходящее или входящее от друга в лс     | `1 << 5`   |
| spam           | Пометка сообщения как спам               | `1 << 6`   |
| deleted        | Удаление сообщения локально              | `1 << 7`   |
| audio_listened | Прослушано голосовое сообщение           | `1 << 12`  |
| chat2          | Отправка в беседу через клиенты          | `1 << 13`  |
| cancel_spam    | Отмена пометки как спам                  | `1 << 15`  |
| hidden         | Приветственное сообщение от группы       | `1 << 16`  |
| deleted_all    | Удаление сообщения для всех              | `1 << 17`  |
| chat_in        | Входящее сообщение в беседе              | `1 << 19`  |
| noname_flag    | Неизвестный флаг                         | `1 << 20`  |
| reply_msg      | Ответ на сообщение                       | `1 << 21`  |

Пример определения флага в маске:
```js
const mask = 1 | 2 | 32;

8 & mask // 0 (false)
2 & mask // 2 (true)
// Однако, если указать в качестве флага не степень двойки,
// то там будет другое число, отличное от 0.
```

### Сервисные сообщения

Сервисное сообщение описывается одним или несколькими ключами в объекте. Тип сервисного сообщения определяет ключ `source_act`.

Возможные значения `source_act`:
- __`chat_photo_update`__ - Обновление фотографии беседы
   - Нет дополнительных ключей, но добавляет фотографию во вложениях
- __`chat_photo_remove`__ - Удаление фотографии беседы
   - Нет дополнительных ключей
- __`chat_create`__ - Создание беседы
   - `source_text` - Название беседы
- __`chat_title_update`__ - Обновление название беседы
   - `source_old_text` - Старое название беседы (но при полчении сообщения из API его не будет)
   - `source_text` - Новое название беседы
- __`chat_pin_message`__ - Закрепление сообщения
   - `source_mid` - `id` закрепившего сообщение
   - `source_message` - обрезанное закрепляемое сообщение
   - `source_chat_local_id` - `conversation_message_id` закрепляемого сообщения
- __`chat_unpin_message`__ - Открепление сообщения
   - `source_mid` - `id` открепившего сообщение
   - `source_chat_local_id` - `conversation_message_id` открепляемого сообщения
- __`chat_invite_user_by_link`__ - Вступление по ссылке
   - Нет дополнительных ключей
- __`chat_invite_user`__ - Вступление или возвращение в беседу
   - `source_mid` - `id` вступившего или вернувшегося
   - Если `source_mid` совпадает с `from`, то он вернулся сам, иначе - его пригласили
- __`chat_kick_user`__ - Исключение или выход из беседы
   - `source_mid` - `id` кикнутого или вышедшего
   - если `source_mid` совпадает с `from`, то он вышел сам, иначе - его кикнули

Пример описания сервисного сообщения:
```js
{
  source_act: 'chat_pin_message',
  source_mid: '88262293',
  source_message: 'Сообщение, которое будет в закрепе',
  source_chat_local_id: '5517'
}
```

### Клавиатура для ботов

Клавиатура для ботов представляет собой обьект с описанием ее типа и кнопок.
Основная структура представлена ниже, остальную информацию можно узнать в [документации](https://vk.com/dev/bots_docs_3?f=4.2.%20%D0%A1%D1%82%D1%80%D1%83%D0%BA%D1%82%D1%83%D1%80%D0%B0%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85).
```js
{
  one_time: true || false, // Скрывать ли при клике на кнопку (не работает для inline)
  inline: true || false, // Клавиатура для сообщения или для беседы
  buttons: Array // Массив с массивами кнопок
}
```
