# Тестовый стенд Auth API

Стенд реализует API из технического задания (версия 1.1): регистрация, вход,
обновление токена, проверка токена и восстановление пароля.

- **Swagger UI** с кнопкой «Try it out»: https://iliavoskoboinikov.github.io/auth-stand-docs/
- **OpenAPI-описание** для импорта в Postman или Insomnia:
  https://iliavoskoboinikov.github.io/auth-stand-docs/openapi.yaml

## Базовый URL

```
https://faiwlkhyssrgofauzegs.supabase.co/functions/v1/
```

```kotlin
Retrofit.Builder()
    .baseUrl("https://faiwlkhyssrgofauzegs.supabase.co/functions/v1/")
```

> **Пути пишите без ведущего слеша:** `@POST("auth/login")`, а не `@POST("/auth/login")`.
> В ТЗ у логина путь написан со слешем, но в коде так нельзя: со слешем Retrofit
> отбросит `/functions/v1/` из `baseUrl`, запрос уйдёт мимо стенда и получит 401
> от шлюза Supabase. Приложение покажет «неверный пароль», хотя дело не в нём.

## Ручки

| Метод и путь             | Тело / заголовки                                  | Успех                                        | Ошибки                                                        |
|--------------------------|---------------------------------------------------|----------------------------------------------|---------------------------------------------------------------|
| `POST auth/registration` | `{"email": String, "password": String}`           | `201` `access_token`, `refresh_token`, `user_id` | `409` пользователь уже существует; `400` некорректный email; `400` пароль менее 7 символов |
| `POST auth/login`        | `{"email": String, "password": String}`           | `200` `access_token`, `refresh_token`, `user_id` | `400` некорректный email; `400` пароль менее 7 символов; `401` неверный email или пароль |
| `POST auth/refresh`      | `{"refresh_token": String}`                       | `200` `access_token`, `refresh_token`         |                                                               |
| `GET auth/check`         | заголовок `Authorization: Bearer <access_token>`  | `200` `{"is_valid": true}`                    |                                                               |
| `POST auth/recovery`     | заголовок `email: <адрес>`                        | `200`                                         | `400` некорректный email                                      |

`user_id` — число (`Long`), токены — строки.

## Что важно знать

1. **`is_valid` — boolean.** Объявляйте поле как `Boolean`, не `String`:
   `@SerialName("is_valid") val isValid: Boolean`.
2. **`auth/check` всегда отвечает 200.** Протухший, неизвестный или отсутствующий
   токен — это `{"is_valid": false}`, а не ошибка. Увидели `false` — делайте
   `auth/refresh`.
3. **`401` в логине — неверный email или пароль.** Обработайте его и покажите
   понятное сообщение.
4. **В `auth/recovery` email передаётся заголовком**, не телом:
   `@POST("auth/recovery") suspend fun recover(@Header("email") email: String)`.
5. **Письма реально не приходят.** Ручка всегда отвечает `200` на корректный email,
   даже если такого пользователя нет.
6. **access-токен живёт 15 минут** — это не баг, а повод реализовать рефреш.
   refresh-токен живёт 30 дней с последнего обновления.
7. **Аккаунты на стенде не удаляются.** Если email занят, зарегистрируйтесь на другой,
   например `ivan+test2@gmail.com`.
8. **В ответе `auth/refresh` нет `user_id`** — храните его с момента входа.
9. **Тело ошибки** — `{"code": "...", "message": "..."}`. По `code` можно различить два
   разных `400`: `INVALID_EMAIL` и `PASSWORD_TOO_SHORT`. Работать только по HTTP-кодам
   тоже можно.

## Отладка рефреша

Чтобы не ждать 15 минут, добавьте к `auth/registration` или `auth/login` заголовок
`X-Debug-Access-Ttl` с числом секунд от 5 до 900. Все access-токены этой сессии,
включая выданные при рефреше, будут жить столько секунд. Удобно подключать
интерсептором только в debug-сборке.
