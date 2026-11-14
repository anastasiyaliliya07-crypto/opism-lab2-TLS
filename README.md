<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->
# Лабораторна робота 2. TLS та сертифікати
| Поле | Значення |
|---|---|
| ПІБ |Назарова Анастасія Олександрівна |
| Група |F5-2.02 |
| Ідентифікатор |cs-202-15 |
| Дата виконання | |
| Середовище | GitHub Codespaces|
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 |
## Параметри варіанта
<!-- Переписати з картки stand/cards/<id>.md. -->
| Параметр | Значення |
|---|---|
| Віддалений сервер |`cdn.cs-202-15.lab` |
| Імена для сертифіката |`www.cs-202-15.lab`, `news.cs-202-15.lab` |
| Ваш сервер (IP) |`10.53.1.1` |
<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->
## Завдання 1. HTTP і HTTPS у дампі трафіку
Вивід кроку 1.1:
```
root@cs-202-15:/lab/work# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://cdn.cs-202-15.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 42
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.cs-202-15.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
09:27:44.304852 IP 10.53.1.1.37628 > 10.53.0.80.80: Flags [P.], seq 1:81, ack 1, win 502, options [nop,nop,TS val 3373508786 ecr 3759236130], length 80: HTTP: GET / HTTP/1.1
......d"GET / HTTP/1.1
Host: cdn.cs-202-15.lab
server: web
Host: cdn.cs-202-15.lab
```
Вивід кроку 1.2:
```
root@cs-202-15:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://cdn.cs-202-15.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'cdn.cs-202-15.lab' t1-https.txt | wc -l
[1] 48
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.cs-202-15.lab
SNI: cdn.cs-202-15.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```
| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` |присутній |зашифровано |
| заголовок `Host` |присутній |зашифровано |
| тіло відповіді (`server: web`) |присутній |зашифровано |
| ім'я віддаленого сервера (кількість входжень) |багато |1 (у SNI) |

**1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?** Ім'я сервера передається у відкритому вигляді в повідомленні ClientHello (це перше повідомлення, яке клієнт надсилає серверу на початку встановлення TLS-з'єднання) у спеціальному розширенні Server Name Indication (SNI).
**2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?** Тому що протокол HTTP працює на транспортному рівні безпосередньо через TCP без жодних шифрованих надбудов, і в ньому взагалі немає поняття SNI (запит іде на конкретну IP-адресу, яку роутер або сервер і так бачить). У HTTPS розширення SNI є обов'язковим: оскільки на одній IP-адресі може працювати безліч різних захищених сайтів, клієнт одразу на початку каже серверу, до якого саме домена він звертається, щоб сервер знав, який саме сертифікат йому показати.
**3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?** Версію HTTP у випадку з HTTPS визначив узгоджений під час TLS-рукостискання протокол за допомогою спеціального розширення ALPN (Application-Layer Protocol Negotiation). Під час рукостискання клієнт пропонує список підтримуваних протоколів (наприклад, h2, http/1.1), а сервер обирає найкращий із них — у вашому випадку це сучасний h2 (HTTP/2). Для звичайного HTTP механізм ALPN відсутній, тому за замовчуванням використовується старіший HTTP/1.1.

## Завдання 2. Рукостискання по записах
Вивід кроку 2.1 (TLS 1.3):
```
root@cs-202-15:/lab/work# openssl s_client -connect cdn.cs-202-15.lab:443 -servername cdn.cs-202-15.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=328
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
    Certificate, Length=983
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
```
root@cs-202-15:/lab/work# openssl s_client -connect cdn.cs-202-15.lab:443 -servername cdn.cs-202-15.lab -alpn h2,http/1.1 \
  -tls1_2 -CAfile $STAND_CA -trace </dev/null > t2-12.txt 2>&1 && \
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt && \
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=223
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=978
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
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```
<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->
| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | | | | | |

Вивід кроків 2.3 і 2.4:
```

```
| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | | |
| Номер першого запису, тип якого спостерігач не бачить | | |
| Кількість RTT від ClientHello до першого HTTP-запиту | | |
| Чи видно сертифікат у дампі (результат 2.4) | | |
| Обраний протокол ALPN | | |
1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
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
