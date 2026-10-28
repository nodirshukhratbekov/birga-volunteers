# Birga — что нужно поменять в Google Apps Script

Сайт (`index.html`) уже обновлён. Пункты 2, 4, 5 работают сразу.
Пункты 1, 3, 6 требуют правок на сервере (Apps Script) — ниже точный формат запросов и ответов, который ждёт сайт.
Ошибки сервер возвращает как и раньше: `{ok:false, error:'e_код'}`, при плохом токене — `{ok:false, code:'auth'}`.

---

## 1. Места на ивенты считаются только после подтверждения

В `getSite` для каждого ивента поле `taken` должно считать регистрации **только** со статусом
`registered` или `paid`:

```js
taken = regs.filter(r => r.eventId === e.id && (r.status === 'registered' || r.status === 'paid')).length;
```

`awaiting_payment`, `pending`, `cancelled`, `rejected` место не занимают.
Пример: 30 мест, админ отметил оплату → у всех показывается «28 из 30».

Рекомендуется: в `setEventRegStatus` при переводе в `paid` (или `approved`) проверять, что `taken < seats`, иначе вернуть `e_full`.

> В админке число мест сайт уже сам считает по списку регистраций — там «28 из 30» видно сразу после нажатия «Отметить оплату».

---

## 3. Смена пароля

```
POST {action:'changePassword', token, oldPassword, newPassword}
```
- `oldPassword` и `newPassword` — SHA-256 (hex), так же, как при `login`/`signup`.
- Проверить `oldPassword` так же, как при логине → если не совпал: `{ok:false, error:'e_pw_wrong'}`.
- Сохранить `newPassword` тем же способом, что и при регистрации.
- Ответ: `{ok:true}`. Можно вернуть новый токен `{ok:true, token:'<64 hex>'}` (если решите сбросить остальные сессии) — сайт его подхватит.
- Желательно ограничить число попыток (как у логина, `e_locked`).

---

## 6. Ивенты с обязательными документами (IELTS, SAT, паспорт)

### Ивент
`addEvent` теперь присылает поле `docs` — массив из `'ielts' | 'sat' | 'passport'` (может быть пустым).
Сохранить (например, строкой `"ielts,sat"`) и отдавать в `getSite` как `docs` (строка через запятую или массив — сайт понимает оба).

### Документы пользователя
Новый лист, например `Documents`: `username | type | fileId | mime | status | uploadedAt`.
Файлы лучше хранить в Google Drive (в ячейку таблицы фото документа не влезет), в закрытой папке.
**Важно:** DriveApp требует нового разрешения — после правок нужно переразвернуть веб-приложение (новая версия) и подтвердить доступ.

```
POST {action:'uploadDocument', token, docType, data}
```
- `docType` ∈ `ielts | sat | passport`
- `data` — `data:image/jpeg;base64,...` или `data:application/pdf;base64,...` (до ~3 МБ; фото сайт уже сжимает)
- Сохранить файл, заменить предыдущий того же типа, статус → `pending`.
- Ответ: `{ok:true, document:{type, status:'pending', uploadedAt}}`
- Ошибка формата: `{ok:false, error:'e_doc_file'}`

`getMyData` дополнительно возвращает:
```
documents: [{type:'ielts', status:'pending'|'approved'|'rejected', uploadedAt}]
```

```
POST {action:'getUserDoc', token, username, docType}   // только админ с правом events или users
```
- Ответ: `{ok:true, data:'data:image/jpeg;base64,...' | 'data:application/pdf;base64,...', status}`
- Нет файла: `{ok:false, error:'e_doc_missing'}`

### Регистрация на такой ивент
`registerEvent` (без изменений в запросе):
- Если у пользователя нет какого-то нужного документа (или он `rejected`) → `{ok:false, error:'e_docs_missing'}`.
- Если все нужные документы уже `approved` → обычная логика (`registered` / `awaiting_payment`).
- Иначе создать регистрацию со статусом `pending` и вернуть `{ok:true, regId, status:'pending'}`.
- Повторная запись разрешена, если прошлая регистрация `cancelled` или `rejected`.

### Проверка админом
`setEventRegStatus` принимает два новых статуса:
- `approved` → документы пользователя для этого ивента помечаются `approved`; регистрация становится `registered` (бесплатный / за лояльность) или `awaiting_payment` (платный).
- `rejected` → регистрация `rejected`; если ивент за лояльность — вернуть баллы (как при отмене); документы можно пометить `rejected`, чтобы человек загрузил заново.

`getEventRegs` — без изменений (статус `pending` сайт сам показывает с кнопками «Одобрить / Отклонить» и «Открыть документ»).

### Безопасность
Паспорт — персональные данные. Папку в Drive не расшаривать, отдавать файлы только через `getUserDoc` с проверкой прав админа.
