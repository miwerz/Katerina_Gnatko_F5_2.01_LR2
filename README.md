# Katerina_Gnatko_F5_2.01_LR2
# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ |Гнатко Катерина Вікторівна |
| Група | F5 2.01 |
| Ідентифікатор |s6 |
| Дата виконання |15.11.2026 |
| Середовище | <!-- GitHub Codespaces або інше; якщо інше, вказати яке --> GitHub Codespaces |
| Версія OpenSSL | <!-- вивід openssl version у терміналі стенда --> |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер |cdn.s6.lab |
| Імена для сертифіката |www.s6.lab, docs.s6.lab |
| Ваш сервер (IP) |10.53.1.1 |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```server: web
Host: cdn.s6.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1

```

Вивід кроку 1.2:

```server: web
Host: cdn.s6.lab
SNI: cdn.s6.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
0
1

```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` |GET / HTTP/1.1 |Зашифровано |
| заголовок `Host` |Host: cdn.s6.lab |Зашифровано |
| тіло відповіді (`server: web`) |присутній (рядок server: web у відповіді) |Зашифровано (або передано у вигляді HTTP/2 зашифрованих фреймів) |
| ім'я віддаленого сервера (кількість входжень) |0 (у транспортному дампі порту 80 ім'я cdn.s6.lab не зустрічається, оскільки використовується лише HTTP-заголовок Host) |1 (зустрічається 1 раз у відкритому полі SNI під час рукостискання TLS) |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS? У першому повідомленні клієнта ClientHello (під час рукостискання TLS) у розширенні server_name (SNI - Server Name Indication).

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS? Тому що для HTTP (порт 80) з'єднання йде по звичайному TCP без TLS-рукостискання, де ім'я передається вже всередині HTTP-заголовка Host. Для HTTPS (порт 443) рукостискання TLS відбувається до встановлення захищеного каналу, і клієнт мусить вказати SNI у відкритому вигляді, щоб сервер зрозумів, який саме сертифікат йому надати.

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку? Механізм ALPN (Application-Layer Protocol Negotiation), який клієнт і сервер успішно узгодили під час безпечного рукостискання TLSv1.3.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=321
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=968
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)

```

Вивід кроку 2.2 (TLS 1.2):

```Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=216
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=963
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)

```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 |К→С |Handshake (22) |- |ClientHello|321 |
| 2 |С→К |Handshake (22) | -|ServerHello | 118|
| 3 |С→К |ChangeCipherSpec (20) |- |ChangeCipherSpec | -|
| 4 |С→К |ApplicationData (23) |Handshake (22) |EncryptedExtensions |15 |
| 5 |С→К |ApplicationData (23) |Handshake (22) |Certificate |968 |
| 6 |С→К |ApplicationData (23) |Handshake (22) |CertificateVerify | 75|
| 7 |С→К |ApplicationData (23) |Handshake (22) |Finished |48 |
| 8 |К→С |ChangeCipherSpec (20) |- |ChangeCipherSpec |- |
| 9 |К→С |ApplicationData (23) |Handshake (22) |Finished |48 |
| 10 |К→С |ApplicationData (23) |Alert (21)|Alert  |- |


Вивід кроків 2.3 і 2.4:

```extension_type=server_name(0), length=15
          0000 - 00 0d 00 00 0a 63 64 6e-2e 73 36 2e 6c 61 62     .....cdn.s6.lab
        extension_type=ec_point_formats(11), length=4
          uncompressed (0)
TLS 1.3: 0
TLS 1.2: 2

```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт |321 |216 |
| Номер першого запису, тип якого спостерігач не бачить |№4 (EncryptedExtensions усередині ApplicationData) |Запис із шифрованим рукостисканням (Finished) |
| Кількість RTT від ClientHello до першого HTTP-запиту |1 RTT |2 RTT|
| Чи видно сертифікат у дампі (результат 2.4) |Ні (0) |Так (2) |
| Обраний протокол ALPN |h2 |http/1.1 (або за замовчуванням) |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису? За появою Content Type = ApplicationData (23) та Inner Content Type = Handshake (22). Спостерігач бачить лише загальний тип ApplicationData, а реальні типи (Certificate, Finished тощо) зашифровані всередині.

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT. Через розширення key_share з ключовими матеріалами для швидкого узгодження (1 RTT).

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені? Складається з: типу (2 б), довжини списку (2 б), типу імені (1 б), довжини домену (2 б) та самого імені. Для cdn.s6.lab (10 символів) довжина поля значення становитиме 15 байт.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює? Заради зворотної сумісності зі старим мережевим обладнанням (middlebox compatibility).

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text

```

Вивід кроку 3.4:

```text

```

Вивід кроку 3.5:

```text

```

Вивід кроку 3.6:

```text

```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | |
| Видавець сертифіката видавця | |
| Імена в SAN | |
| Строк дії (від – до) | |
| Тип ключа сервера і алгоритм підпису | |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text

```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | | | | | |
| f2 | | | | | |
| f3 | | | | | |
| f4 | | | | | |
| f5 | | | | | |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | | | |
| `time_appconnect` | | | |
| `time_starttransfer` | | | |

Вивід кроків 5.2 і 5.3:

```text

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | | | | | | |
| TLS 1.2, нове | | | | | | |
| 5.3, перший запит | | | | | | |
| 5.3, другий запит | | | | | | |
| 5.3, третій запит | | | | | | |

RTT, визначений вимірюванням, і з якого значення його визначено:

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
