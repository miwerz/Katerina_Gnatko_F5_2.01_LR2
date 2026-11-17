# Katerina_Gnatko_F5_2.01_LR2
# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ |Гнатко Катерина Вікторівна |
| Група | F5 2.01 |
| Ідентифікатор |s6 |
| Дата виконання |15.11.2026 |
| Середовище | <!-- GitHub Codespaces або інше; якщо інше, вказати яке --> GitHub Codespaces |
| Версія OpenSSL | <!-- вивід openssl version у терміналі стенда --> OpenSSL 3.0.2 15 Mar 2022|

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

```збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 27a0d7
notBefore=Nov 17 18:44:50 2026 GMT
notAfter=Nov 17 18:44:50 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```2026/11/17 18:46:42 [INFO] [www.s6.lab, docs.s6.lab] acme: Validations succeeded; requesting certificates
2026/11/17 18:46:42 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/17 18:46:42 [INFO] [www.s6.lab] Server responded with a certificate.

10.53.0.10 - - [17/Nov/2026:18:46:35 +0000] "GET /.well-known/acme-challenge/_h-OIwovQPqcf7ajdT6Ondc1uTatw4PeUWFovxTFcW0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:18:46:39 +0000] "GET /.well-known/acme-challenge/ZJFReTzAUt5H9eYbWojznopDkkxC0jSmrFZSV4_x4Bk HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"

```

Вивід кроку 3.4:

```subject=CN = www.s6.lab
issuer=CN = Pebble Intermediate CA 53cbe4
notBefore=Nov 17 18:46:42 2026 GMT
notAfter=Nov 17 18:46:41 2031 GMT
serial=2C00C778A1E668D3
X509v3 Subject Alternative Name: 
    DNS:www.s6.lab, DNS:docs.s6.lab
subject=CN = Pebble Intermediate CA 53cbe4
issuer=CN = Pebble Root CA 27a0d7

```

Вивід кроку 3.5:

```Certificate chain
 0 s:CN = www.s6.lab
   i:CN = Pebble Intermediate CA 53cbe4
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 18:46:42 2026 GMT; NotAfter: Nov 17 18:46:41 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)

```

Вивід кроку 3.6:

```Certificate chain
 0 s:CN = www.s6.lab
   i:CN = Pebble Intermediate CA 53cbe4
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 18:46:42 2026 GMT; NotAfter: Nov 17 18:46:41 2031 GMT
 1 s:CN = www.s6.lab
   i:CN = Pebble Intermediate CA 53cbe4
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 18:44:50 2026 GMT; NotAfter: Nov 17 18:44:50 2056 GMT
---
Verify return code: 0 (ok)
server: s6 (https)
Host: docs.s6.lab
SNI: docs.s6.lab
protocol: TLSv1.3

```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера |CN = Pebble Intermediate CA 53cbe4 |
| Видавець сертифіката видавця |CN = Pebble Root CA 27a0d7 |
| Імена в SAN |DNS:www.s6.lab, DNS:docs.s6.lab |
| Строк дії (від – до) |Nov 17 18:46:42 2026 GMT – Nov 17 18:46:41 2031 GMT |
| Тип ключа сервера і алгоритм підпису | id-ecPublicKey (ECDSA) / RSA-SHA256|
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 |У кроці 3.5: 1; у кроці 3.6: 2 |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.   Перевірку здійснив ACME-сервер (Pebble VA) за допомогою протоколу HTTP-01. Це підтверджують такі рядки виводу: [INFO] [docs.s6.lab] acme: use http-01 solver
[INFO] [www.s6.lab] acme: use http-01 solver
[INFO] [docs.s6.lab] The server validated our request
[INFO] [www.s6.lab] The server validated our request

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?  Сервер надіслав клієнту лише власний кінцевий сертифікат (www.s6.lab.crt), але не передав проміжний сертифікат (Intermediate CA). Хоча корінь був відомий через --cacert, клієнту не вистачило проміжної ланки, щоб побудувати безперервний ланцюжок довіри (через що виникла помилка unable to verify the first certificate).

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?  id-ecPublicKey - це тип відкритого ключа самого сервера, який використовує криптографію на еліптичних кривих (ECDSA). RSA-SHA256 - це алгоритм, за допомогою якого Центр Сертифікації (CA) підписав цей сертифікат (використовуючи власний RSA-ключ CA). Вони не суперечать одне одному, оскільки криптографічний алгоритм ключа власника сертифіката й алгоритм цифрового підпису видавця (CA) можуть відрізнятися.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?  Перевірка зламається, оскільки тестовий ACME-сервер (Pebble) є ефемерним і при повному перезапуску стенда втрачає свій стан або згенеровані сертифікати в робочій директорії. У реальній інфраструктурі цьому відповідає ситуація, коли сертифікат на сервері було випадково видалено, пошкоджено його файли або втрачено приватний ключ.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```=== SERVER: f1.s6.lab ===
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.s6.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: May  1 18:44:48 2026 GMT; NotAfter: Oct 28 18:44:48 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Nov 16 18:44:48 2031 GMT
---
Verify return code: 10 (certificate has expired)

=== SERVER: f2.s6.lab ===
server: web, f2
Certificate chain
 0 s:CN = f2.s6.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Feb 15 18:44:48 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Nov 16 18:44:48 2031 GMT
---
Verify return code: 0 (ok)

=== SERVER: f3.s6.lab ===
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f3.s6.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Feb 15 18:44:48 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Nov 16 18:44:48 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)

=== SERVER: f4.s6.lab ===
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.s6.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Feb 15 18:44:48 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)

=== SERVER: f5.s6.lab ===
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f5.s6.lab
   i:CN = f5.s6.lab
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Nov 17 18:44:48 2026 GMT; NotAfter: Feb 15 18:44:48 2027 GMT
---
Verify return code: 18 (self-signed certificate)

```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | SSL certificate problem: certificate has expired|10 (certificate has expired) |Перевірка терміну дії сертифікату |NotAfter: Oct 28 18:44:48 2026 GMT / Verify return code: 10 |Перевипустити та оновити сертифікат сервера |
| f2 |Відсутня (успішно) |0 (ok) |Перевірка успішна (немає помилок) |Verify return code: 0 (ok) |Нічого (конфігурація правильна) |
| f3 |SSL certificate problem: unable to get local issuer certificate |20 (unable to get local issuer certificate) |Довіра до видавця (сторонній CA відсутній у довірених)|i:O = Foreign Org, CN = Foreign Intermediate CA / Verify return code: 20|Додати кореневий сертифікат стороннього CA до сховища довірених або змінити CA |
| f4 |SSL certificate problem: unable to get local issuer certificate |21 (unable to verify the first certificate) |Перевірка ланцюжка (неповний ланцюжок, відсутній Intermediate CA)) |У ланцюжку лише 1 сертифікат (0 s:CN = f4.s6.lab) / Verify return code: 21 |Налаштувати вебсервер (Nginx) на передачу повного ланцюжка (кінцевий + проміжний) |
| f5 |SSL certificate problem: self-signed certificate |18 (self-signed certificate) |Перевірка на самопідписаний сертифікат |s:CN = f5.s6.lab і i:CN = f5.s6.lab (збігаються) / Verify return code: 18 |Отримати та встановити дійсний сертифікат, виданий легітимним Центром Сертифікації |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`? Тут є обидва варіанти: на f3 сертифікат виданий чужим центром (Foreign Org), якому клієнт не довіряє, а f4 просто забув надіслати проміжний сертифікат. У виводі openssl s_client це легко відрізнити: для f3 у блоці Certificate chain видно два сертифікати (кінцевий і проміжний) і код помилки 20, а для f4 - лише один сертифікат самого сервера та код 21.

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.  openssl s_client за замовчуванням перевіряє лише сам ланцюжок підписів і дату дії, але не звіряє доменне ім'я сайту з полем SAN у сертифікаті (тому й пише 0 (ok)). А curl обов'язково перевіряє, чи збігається адреса сайту з ім'ям у сертифікаті, і якщо ні - блокує з'єднання.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?  З ключем curl -k відповіли б усі п'ять серверів (f1–f5), бо цей прапорець повністю вимикає перевірку сертифікатів. Порівняно з --cacert ми при цьому втрачаємо автентифікацію сервера: трафік хоч і шифрується, але ми не можемо бути впевнені, що спілкуємося зі справжнім сервером, а не з перехоплювачем (атака MitM).

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` |1 RTT |1 RTT |0 RTT |
| `time_appconnect` |2 RTT (1 TCP + 1 TLS) | 3 RTT (1 TCP + 2 TLS)|0 RTT |
| `time_starttransfer` |3 RTT (1 TCP + 1 TLS + 1 HTTP) |4 RTT (1 TCP + 2 TLS + 1 HTTP) |1 RTT (лише HTTP) |

Вивід кроків 5.2 і 5.3:

```=== 5.2 TLS 1.3 (3 спроби) ===
nl=0.080983 conn=0.121178 app=0.166044 ttfb=0.206430 tot=0.206491 n=1
nl=0.040473 conn=0.080611 app=0.124390 ttfb=0.165135 tot=0.165306 n=1
nl=0.040649 conn=0.080794 app=0.125393 ttfb=0.165973 tot=0.166037 n=1

=== 5.2 TLS 1.2 (3 спроби) ===
nl=0.041674 conn=0.081819 app=0.165737 ttfb=0.206121 tot=0.206254 n=1
nl=0.040614 conn=0.080768 app=0.165459 ttfb=0.205849 tot=0.205941 n=1
nl=0.040480 conn=0.080639 app=0.165853 ttfb=0.206244 tot=0.206319 n=1

=== 5.3 Повторне використання з'єднання ===
nl=0.040532 conn=0.080683 app=0.125232 ttfb=0.165499 tot=0.165562 n=1
nl=0.000018 conn=0.000000 app=0.000000 ttfb=0.040255 tot=0.040292 n=0
nl=0.000014 conn=0.000000 app=0.000000 ttfb=0.040357 tot=0.040395 n=0

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове |40.5 ms |80.8 ms |124.4 ms |165.9 ms |166.0 ms |1 |
| TLS 1.2, нове |40.6 ms |80.8 ms |165.5 ms |205.8 ms |205.9 ms |1 |
| 5.3, перший запит |40.5 ms |80.7 ms |125.2 ms |165.5 ms |165.6 ms |1 |
| 5.3, другий запит | 0.02 ms|0.0 ms |0.0 ms |40.3 ms |40.3 ms |0 |
| 5.3, третій запит |0.01 ms |0.0 ms | 0.0 ms|40.4 ms |40.4 ms |0 |

RTT, визначений вимірюванням, і з якого значення його визначено:  40 мс (визначено з часу starttransfer повторного запиту в кроці 5.3 - 40.3 мс, а також із різниці між етапами connect і namelookup: 80.8 − 40.5 ≈ 40.3 мс).

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?  TLS 1.3 (нове): namelookup - 1 RTT, connect - 2 RTT, appconnect - 3 RTT, starttransfer - 4 RTT.
TLS 1.2 (нове): namelookup - 1 RTT, connect - 2 RTT, appconnect - 4 RTT, starttransfer - 5 RTT.
Повторний запит: namelookup/connect/appconnect - 0 RTT, starttransfer - 1 RTT. Чому значення трохи більші за ціле число RTT: Додається приблизно 4–6 мс на роботу процесора під час TLS-рукостискання (генерація ключів, шифрування та перевірка підписів сертифікатів).

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?  Сам клієнт (curl) не зберігає DNS-кеш між запусками, тому щоразу по мережі звертається до резолвера, витрачаючи 1 RTT (~40 мс). Вдвічі більшим (~80 мс) він був би, якби кеш резолвера був порожнім (як у першому вимірюванні 81 мс) і йому довелося б робити додатковий запит до авторитетного DNS-сервера (+1 RTT).

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.
3. Розрахунок часу очікування мережі для 200 запитів ($1\text{ RTT} \approx 40\text{ мс}$):

* **TLS 1.3 (нові з'єднання):** $200 \times 3\text{ RTT} = 600\text{ RTT} \approx \mathbf{24\text{ с}}$ (або ~33.2 с з урахуванням первинного DNS-запиту на кожне з'єднання).
* **TLS 1.2 (нові з'єднання):** $200 \times 4\text{ RTT} = 800\text{ RTT} \approx \mathbf{32\text{ с}}$ (або ~41.2 с з урахуванням DNS).
* **Через одне спільне з'єднання:** $4\text{ RTT}$ (перший запит) $+ 199 \times 1\text{ RTT} = 203\text{ RTT} \approx \mathbf{8.1\text{ с}}$ (у 4–5 разів швидше завдяки Keep-Alive).

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно: 

Незважаючи на те, що сертифікат було успішно випущено через ACME-сервер, а правильний корінь довіри вказано через `--cacert`, перша перевірка з'єднання завершилася невдало, оскільки вебсервер за замовчуванням передавав лише кінцевий сертифікат без проміжного (Intermediate CA).

*Рядок виводу:*
```text
Verify return code: 21 (unable to verify the first certificate)

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:  Чому клієнт `lego` спочатку намагається знайтиsolver для `tls-alpn-01`, хоча ми явно вказуємо параметр використання вебруту для HTTP-01. Рядок виводу: `[INFO] [docs.s6.lab] acme: Could not find solver for: tls-alpn-01` з наступним переходом на `use http-01 solver`. Що вже перевірено: Вебрут і порт налаштовані правильно, валідація через HTTP-01 у підсумку завершується успішно.

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні --> так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 |Gemini |допоможи та поясни покроково як бідняти стенд |допоміг підняти стенд, та потім ще пояснював для чого які команди |
| 2 |Gemini |чому у деяких виводах було нот фаунд і це було нормально | пояснив |
| 3 |Gemini |напиши пояснення термінології яка тут використовується коротко та простими словами | допоміг зрозуміти значення |
| 4 |Gemini |сформулюй коротшу та структуровану відповідб яку я зможу гарненько вставити у гітхаб |у 5 завданні допоміг сформулювати відповідь на 3 питання, щоб воно нормально виглядало у гітхабі |
