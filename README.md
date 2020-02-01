# User Long Poll API

### План документации:
1. [Подключение](https://github.com/danyadev/longpoll-doc#подключение)
2. [Возвращаемые ошибки](https://github.com/danyadev/longpoll-doc#возвращаемые-ошибки)
3. [Структура событий](https://github.com/danyadev/longpoll-doc#структура-событий)
 * а
 * б
 * в
4. [Дополнительные данные](https://github.com/danyadev/longpoll-doc#дополнительные-данные)
 * Флаги сообщений
 * Вложения
 * Идентификаторы платформ

Документация написана для __11__ версии Long Poll.

## Подключение

Long Polling - это способ получения событий в реальном времени используя бесконечную цепочку запрос - ответ - запрос... При подаче запроса сервер возвращает ответ не сразу, а когда придет новое событие (либо истечет время ожидания).

Для запроса нужно использовать следующую ссылку:

> https://**`server`**?act=a_check&key=**`key`**&ts=**`ts`**&wait=**`wait`**&mode=**`mode`**&version=**`version`**

* `server`, `key` и `ts` получаются методом [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer)
* `version` - Версия Long Poll
* `wait` - Время ожидания нового события в секундах
* `mode` - Дополнительные опции ответа:
  * `2` - Возвращать вложения
  * `8` - Возвращать нормальные данные некоторых событий
  * `32` - Возвращать `pts`
  * `64` - Возвращать данные о платформе в событии онлайна юзера
  * `128` - Возвращать `random_id`

В `Node.js` ссылку можно составить следующим образом:
```js
// server, key и ts нужно заранее получить. 
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

## Возвращаемые ошибки

## Структура событий

## Дополнительные данные
