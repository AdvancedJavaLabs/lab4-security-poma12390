# Этап 3 — Ручное тестирование

## Условия выполнения

| Поле | Значение |
|---|---|
| Дата тестирования | 2026-05-11 |
| База URL | `http://localhost:7000` |
| Инструмент | `curl` |
| Сырой лог | `artifacts/03_manual_testing_raw.log` |

## `POST /register`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| R8 | Позитив: регистрация нового пользователя | `curl -sS -X POST --get --data-urlencode "userId=fresh_user" --data-urlencode "userName=Fresh" "http://localhost:7000/register"` | 200 | `User registered: true` | Базовая регистрация работает. |
| R2 | Граница: повторная регистрация того же `userId` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "userName=Alice2" "http://localhost:7000/register"` | 200 | `User registered: false` | Дубликат отклоняется, но возвращается 200 с флагом в теле. |
| R3 | Негатив: отсутствует `userName` | `curl -sS -X POST --get --data-urlencode "userId=missing_name" "http://localhost:7000/register"` | 400 | `Missing parameters` | Есть базовая валидация обязательных параметров. |
| R4 | Security: ввод XSS-пейлоада в `userName` | `curl -sS -X POST --get --data-urlencode "userId=xss_user" --data-urlencode "userName=<script>alert(1)</script>" "http://localhost:7000/register"` | 200 | `User registered: true` | Ввод со скриптом сохраняется без фильтрации. |
| R5 | Security: спецсимволы `< > " ' / ..` в `userName` | `curl -sS -X POST --get --data-urlencode "userId=special_chars" --data-urlencode "userName=<b>\"' / ..</b>" "http://localhost:7000/register"` | 200 | `User registered: true` | Спецсимволы принимаются и сохраняются. |

## `POST /recordSession`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| S1 | Позитив: корректные ISO-времена | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "loginTime=2026-05-10T10:00:00" --data-urlencode "logoutTime=2026-05-10T11:30:00" "http://localhost:7000/recordSession"` | 200 | `Session recorded` | Валидная сессия сохраняется. |
| S2 | Негатив: неверный формат `loginTime` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "loginTime=bad-time" --data-urlencode "logoutTime=2026-05-10T11:30:00" "http://localhost:7000/recordSession"` | 400 | `Invalid data: Text 'bad-time' could not be parsed` | Формат даты/времени валидируется. |
| S3 | Негатив: отсутствует `logoutTime` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "loginTime=2026-05-10T10:00:00" "http://localhost:7000/recordSession"` | 400 | `Missing parameters` | Обязательные параметры проверяются. |
| S4 | Негатив: несуществующий пользователь | `curl -sS -X POST --get --data-urlencode "userId=ghost" --data-urlencode "loginTime=2026-05-10T10:00:00" --data-urlencode "logoutTime=2026-05-10T11:30:00" "http://localhost:7000/recordSession"` | 404 | `User not found` | Для неизвестного `userId` запись отклоняется. |
| S6 | Security/граница: `logoutTime < loginTime` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "loginTime=2026-05-10T12:00:00" --data-urlencode "logoutTime=2026-05-10T11:00:00" "http://localhost:7000/recordSession"` | 200 | `Session recorded` | Нелогичная сессия принимается, контроль порядка времени отсутствует. |

## `GET /totalActivity`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| T1 | Позитив: активность существующего пользователя | `curl -sS -G --data-urlencode "userId=u_normal" "http://localhost:7000/totalActivity"` | 200 | `Total activity: 90 minutes` | Расчёт работает для валидных данных. |
| T2 | Граница после tampering-ввода | `curl -sS -G --data-urlencode "userId=u_normal" "http://localhost:7000/totalActivity"` | 200 | `Total activity: 30 minutes` | Принятая отрицательная сессия уменьшила итог, что подтверждает проблему целостности. |
| T3 | Негатив: отсутствует `userId` | `curl -sS -G "http://localhost:7000/totalActivity"` | 400 | `Missing userId` | Есть базовая проверка параметра. |
| T4 | Граница: неизвестный `userId` | `curl -sS -G --data-urlencode "userId=ghost" "http://localhost:7000/totalActivity"` | 200 | `Total activity: 0 minutes` | Для несуществующего пользователя возвращается 0, а не ошибка. |

## `GET /inactiveUsers`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| I1 | Позитив: получить список при `days=30` | `curl -sS -G --data-urlencode "days=30" "http://localhost:7000/inactiveUsers"` | 500 | `It looks like you don't have an object mapper configured` | Возврат JSON падает из-за отсутствия object mapper. |
| I2 | Негатив: отсутствует `days` | `curl -sS -G "http://localhost:7000/inactiveUsers"` | 400 | `Missing days parameter` | Проверка обязательного параметра есть. |
| I3 | Негатив: нечисловой `days` | `curl -sS -G --data-urlencode "days=abc" "http://localhost:7000/inactiveUsers"` | 400 | `Invalid number format for days` | Формат числа валидируется. |
| I4 | Security/граница: отрицательный `days=-1` | `curl -sS -G --data-urlencode "days=-1" "http://localhost:7000/inactiveUsers"` | 500 | `It looks like you don't have an object mapper configured` | Ветка с JSON также падает 500; корректность обработки отрицательной границы не проверить из-за ошибки сериализации. |

## `GET /monthlyActivity`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| M1 | Позитив: метрика за валидный месяц | `curl -sS -G --data-urlencode "userId=u_normal" --data-urlencode "month=2026-05" "http://localhost:7000/monthlyActivity"` | 400 | `It looks like you don't have an object mapper configured` | Позитивный JSON-ответ недоступен из-за отсутствия object mapper. |
| M2 | Негатив: отсутствует `month` | `curl -sS -G --data-urlencode "userId=u_normal" "http://localhost:7000/monthlyActivity"` | 400 | `Missing parameters` | Проверка обязательных параметров есть. |
| M3 | Негатив: неверный формат месяца | `curl -sS -G --data-urlencode "userId=u_normal" --data-urlencode "month=2026/05" "http://localhost:7000/monthlyActivity"` | 400 | `Text '2026/05' could not be parsed at index 4` | Формат `yyyy-MM` валидируется. |
| M4 | Граница: пользователь без сессий | `curl -sS -G --data-urlencode "userId=no_session_user" --data-urlencode "month=2026-05" "http://localhost:7000/monthlyActivity"` | 400 | `Invalid data: No sessions found for user` | Для пользователя без сессий возвращается ошибка бизнес-логики. |

## `GET /userProfile`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| U1 | Позитив: профиль существующего пользователя | `curl -sS -G --data-urlencode "userId=u_normal" "http://localhost:7000/userProfile"` | 200 | `<h1>Profile: Alice</h1>` | HTML-профиль возвращается корректно. |
| U2 | Негатив: отсутствует `userId` | `curl -sS -G "http://localhost:7000/userProfile"` | 400 | `Missing userId` | Проверка обязательного параметра есть. |
| U3 | Негатив: неизвестный `userId` | `curl -sS -G --data-urlencode "userId=ghost" "http://localhost:7000/userProfile"` | 404 | `User not found` | Для неизвестного пользователя возвращается 404. |
| U4 | Security: XSS-пейлоад в HTML без экранирования | `curl -sS -G --data-urlencode "userId=xss_user" "http://localhost:7000/userProfile"` | 200 | `<h1>Profile: <script>alert(1)</script></h1>` | Подтверждён reflected/stored XSS-вектор. |
| U5 | Security: отражение `< > " ' / ..` | `curl -sS -G --data-urlencode "userId=special_chars" "http://localhost:7000/userProfile"` | 200 | `<h1>Profile: <b>"' / ..</b></h1>` | HTML-спецсимволы отдаются без нейтрализации. |

## `GET /exportReport`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| E1 | Позитив: экспорт в обычный файл | `curl -sS -G --data-urlencode "userId=u_normal" --data-urlencode "filename=report.txt" "http://localhost:7000/exportReport"` | 200 | `Report saved to: /tmp/reports/report.txt` | Экспорт работает. |
| E2 | Негатив: отсутствует `filename` | `curl -sS -G --data-urlencode "userId=u_normal" "http://localhost:7000/exportReport"` | 400 | `Missing parameters` | Проверка обязательных параметров есть. |
| E3 | Негатив: неизвестный `userId` | `curl -sS -G --data-urlencode "userId=ghost" --data-urlencode "filename=ghost.txt" "http://localhost:7000/exportReport"` | 404 | `User not found` | Для неизвестного пользователя экспорт не выполняется. |
| E4 | Security: path traversal через `../` | `curl -sS -G --data-urlencode "userId=u_normal" --data-urlencode "filename=../escape.txt" "http://localhost:7000/exportReport"` | 200 | `Report saved to: /tmp/reports/../escape.txt` | Путь не нормализуется/не фильтруется, traversal-вектор подтверждён. |

## `POST /notify`

| Кейc | Цель | curl-запрос | HTTP-код | Важный фрагмент ответа | Вывод |
|---|---|---|---|---|---|
| N1 | Позитив: callback на локальный эндпоинт | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "callbackUrl=http://localhost:7000/totalActivity?userId=u_normal" "http://localhost:7000/notify"` | 200 | `Notification sent. Response: Total activity: 30 minutes` | Внешний запрос по переданному URL выполняется. |
| N2 | Негатив: отсутствует `callbackUrl` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" "http://localhost:7000/notify"` | 400 | `Missing parameters` | Проверка обязательных параметров есть. |
| N3 | Негатив: неизвестный `userId` | `curl -sS -X POST --get --data-urlencode "userId=ghost" --data-urlencode "callbackUrl=http://localhost:7000/totalActivity?userId=u_normal" "http://localhost:7000/notify"` | 404 | `User not found` | Для неизвестного пользователя операция блокируется. |
| N4 | Негатив/security: невалидный URL | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "callbackUrl=not-a-url" "http://localhost:7000/notify"` | 500 | `Notification failed: no protocol: not-a-url` | Ошибка URL обрабатывается, но возвращается 500 с деталями исключения. |
| N5 | Security: SSRF к внутреннему endpoint (`/userProfile`) | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "callbackUrl=http://localhost:7000/userProfile?userId=xss_user" "http://localhost:7000/notify"` | 200 | `Notification sent. Response: <html>...<script>alert(1)</script>...` | Сервер ходит по произвольному URL и возвращает контент вызывающей стороне. |
| N6 | Security: чтение локального файла через `file://` | `curl -sS -X POST --get --data-urlencode "userId=u_normal" --data-urlencode "callbackUrl=file:///etc/hosts" "http://localhost:7000/notify"` | 200 | `Notification sent. Response: ... localhost ...` | Подтверждён доступ к локальным файлам через URL-схему `file://`. |
