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
| Дата виконання |14.11.2026 |
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
| ім'я віддаленого сервера (кількість входжень) |3 (зустрічається у виводі curl та двічі у перехопленому пакеті HTTP: у запиті та у відповіді сервера) |1 (у SNI) |

**1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?** 

Ім'я сервера передається у відкритому вигляді в повідомленні ClientHello (це перше повідомлення, яке клієнт надсилає серверу на початку встановлення TLS-з'єднання) у спеціальному розширенні Server Name Indication (SNI).

**2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?** 

Тому що протокол HTTP працює на транспортному рівні безпосередньо через TCP без жодних шифрованих надбудов, і в ньому взагалі немає поняття SNI (запит іде на конкретну IP-адресу, яку роутер або сервер і так бачить). У HTTPS розширення SNI є обов'язковим: оскільки на одній IP-адресі може працювати безліч різних захищених сайтів, клієнт одразу на початку каже серверу, до якого саме домена він звертається, щоб сервер знав, який саме сертифікат йому показати.

**3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?** 

Версію HTTP у випадку з HTTPS визначив узгоджений під час TLS-рукостискання протокол за допомогою спеціального розширення ALPN (Application-Layer Protocol Negotiation). Під час рукостискання клієнт пропонує список підтримуваних протоколів (наприклад, h2, http/1.1), а сервер обирає найкращий із них — у моєму випадку це сучасний h2 (HTTP/2). Для звичайного HTTP механізм ALPN відсутній, тому за замовчуванням використовується старіший HTTP/1.1.

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
| 1 |К→С |Handshake (22) |- |ClientHello |328 |
|2 |С→К |Handshake (22) |- |ServerHello |118 |
|3 |С→К |ChangeCipherSpec (20) |- |(ChangeCipherSpec) |- |
|4 |С→К |ApplicationData (23) |Handshake (22) |EncryptedExtensions |15 |
|5 |С→К |ApplicationData (23) |Handshake (22) |Certificate |983 |
|6 |С→К |ApplicationData (23) |Handshake (22) |CertificateVerify |75 |
|7 |С→К|ApplicationData (23) |Handshake (22) |Finished |48 |
|8 |К→С |ChangeCipherSpec (20) |- |(ChangeCipherSpec) |- |
|9 |К→С |ApplicationData (23) |Handshake (22) |Finished |48 |
|10 |К→С |ApplicationData (23) |Alert (21) |(Alert) |- |

Вивід кроків 2.3 і 2.4:
```
root@cs-202-15:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=22
          0000 - 00 14 00 00 11 63 64 6e-2e 63 73 2d 32 30 32   .....cdn.cs-202
          000f - 2d 31 35 2e 6c 61 62                           -15.lab
        extension_type=ec_point_formats(11), length=4

root@cs-202-15:/lab/work# for v in 1.3 1.2; do
  timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://cdn.cs-202-15.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 65
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 71
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```
| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт |328 |223 |
| Номер першого запису, тип якого спостерігач не бачить |4 (EncryptedExtensions) |У TLS 1.2 заголовок `Content Type` не шифрується (спостерігач бачить типи всіх записів), а вміст починає шифруватися з запису № 8 (Finished) |
| Кількість RTT від ClientHello до першого HTTP-запиту |1 |2 |
| Чи видно сертифікат у дампі (результат 2.4) |Ні (0) |Так (2) |
| Обраний протокол ALPN |h2 |h2 |

**1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?** 

Шифрування починається з моменту, коли Content Type змінюється на ApplicationData (23), а справжній тип переноситься в Inner Content Type (наприклад, Inner Content Type = Handshake (22)). Спостерігач на мережевому рівні бачить лише загальний тип ApplicationData, а внутрішні типи та вміст пакетів зашифровані.

**2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.**

ClientHello у TLS 1.3 містить розширення для попереднього обміну та вибору криптографічних параметрів (key_share, а також підтримку попереднього використання сесій — PSK/Early Data), що дозволяє клієнту одразу надіслати ключі. Це розширення називається key_share (або розширення попереднього обміну), і саме воно скорочує рукостискання до 1 RTT.

**3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?** 

Значення складається з: 
2 байти — загальна довжина списку імен (00 14 = 20 байт);
1 байт — тип імені (00 — host_name);
2 байти — довжина самого доменного імені (00 11 = 17 байт);
безпосередньо рядок доменного імені у кодуванні ASCII.

Для мого імені cdn.cs-202-15.lab (довжина 17 символів = 17 байт) загальна довжина розширення становить: 
2+1+2+17=22 байти, що підтверджується рядком виводу extension_type=server_name(0), length=22.

**4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?**

Він передається виключно для забезпечення зворотної сумісності (middlebox compatibility) зі старими мережевими пристроями, які очікують побачити цей байт у протоколі, хоча криптографічно в TLS 1.3 він не використовується і нічого не змінює.

## Завдання 3. Сертифікат для власного сервера
Вивід кроків 3.1 і 3.2:
```
root@cs-202-15:/lab/work# acme-root
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 697adc
notBefore=Nov 14 08:58:23 2026 GMT
notAfter=Nov 14 08:58:23 2056 GMT

root@cs-202-15:/lab/work# curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-202-15.lab/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```
Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):
```
root@cs-202-15:/lab/work# lego --server $ACME_DIR --email admin@cs-202-15.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.cs-202-15.lab --domains news.cs-202-15.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/14 10:25:24 No key found for account admin@cs-202-15.lab. Generating a P256 key.
2026/11/14 10:25:24 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@cs-202-15.lab/keys/admin@cs-202-15.lab.key
2026/11/14 10:25:24 [INFO] acme: Registering account for admin@cs-202-15.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/14 10:25:24 [INFO] [www.cs-202-15.lab, news.cs-202-15.lab] acme: Obtaining SAN certificate
2026/11/14 10:25:24 [INFO] [news.cs-202-15.lab] AuthURL: https://ca.lab:14000/authZ/jT9djhCbJ2M99tptCe2k7uf1Vox2uTUAFBtImfS5j4c
2026/11/14 10:25:24 [INFO] [www.cs-202-15.lab] AuthURL: https://ca.lab:14000/authZ/469cQyx9R_l-O05-y4swXhjnxDFnOzcn-ZZ5MDbjW4s
2026/11/14 10:25:24 [INFO] [www.cs-202-15.lab] acme: Could not find solver for: tls-alpn-01
2026/11/14 10:25:24 [INFO] [www.cs-202-15.lab] acme: use http-01 solver
2026/11/14 10:25:24 [INFO] [news.cs-202-15.lab] acme: Could not find solver for: tls-alpn-01
2026/11/14 10:25:24 [INFO] [news.cs-202-15.lab] acme: use http-01 solver
2026/11/14 10:25:24 [INFO] [www.cs-202-15.lab] acme: Trying to solve HTTP-01
2026/11/14 10:25:30 [INFO] [www.cs-202-15.lab] The server validated our request
2026/11/14 10:25:30 [INFO] [news.cs-202-15.lab] acme: Trying to solve HTTP-01
2026/11/14 10:25:34 [INFO] [news.cs-202-15.lab] The server validated our request
2026/11/14 10:25:34 [INFO] [www.cs-202-15.lab, news.cs-202-15.lab] acme: Validations succeeded; requesting certificates
2026/11/14 10:25:34 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/14 10:25:35 [INFO] [www.cs-202-15.lab] Server responded with a certificate.
10.53.0.10 - - [14/Nov/2026:10:25:24 +0000] "GET /.well-known/acme-challenge/b_Z4-F6ilJa-zAUZ-WGklv7n039hkSRtStIaCF25_G0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:10:25:24 +0000] "GET /.well-known/acme-challenge/b_Z4-F6ilJa-zAUZ-WGklv7n039hkSRtStIaCF25_G0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:10:25:24 +0000] "GET /.well-known/acme-challenge/b_Z4-F6ilJa-zAUZ-WGklv7n039hkSRtStIaCF25_G0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:10:25:30 +0000] "GET /.well-known/acme-challenge/ZtS3tLUbkRwtaV6ua0UykC8X0fZMMyjpaGmudnhg1Ug HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:10:25:30 +0000] "GET /.well-known/acme-challenge/ZtS3tLUbkRwtaV6ua0UykC8X0fZMMyjpaGmudnhg1Ug HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [14/Nov/2026:10:25:30 +0000] "GET /.well-known/acme-challenge/ZtS3tLUbkRwtaV6ua0UykC8X0fZMMyjpaGmudnhg1Ug HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```
Вивід кроку 3.4:
```
root@cs-202-15:/lab/work# cd /lab/work/lego/certificates
openssl x509 -in www.cs-202-15.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName 
openssl x509 -in www.cs-202-15.lab.issuer.crt -noout -subject -issuer
subject=CN = www.cs-202-15.lab
issuer=CN = Pebble Intermediate CA 7d35d9
notBefore=Nov 14 10:25:34 2026 GMT
notAfter=Nov 14 10:25:33 2031 GMT
serial=193A6D90558122DB
X509v3 Subject Alternative Name: 
    DNS:www.cs-202-15.lab, DNS:news.cs-202-15.lab
subject=CN = Pebble Intermediate CA 7d35d9
issuer=CN = Pebble Root CA 697adc
```
Вивід кроку 3.5:
```
root@cs-202-15:/lab/work/lego/certificates# cp www.cs-202-15.lab.crt /lab/tls/server.crt
cp www.cs-202-15.lab.key /lab/tls/server.key
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-202-15.lab/
openssl s_client -connect www.cs-202-15.lab:443 -servername www.cs-202-15.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/14 10:31:07 [notice] 94#94: signal process started
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.cs-202-15.lab
   i:CN = Pebble Intermediate CA 7d35d9
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 14 10:25:34 2026 GMT; NotAfter: Nov 14 10:25:33 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```
Вивід кроку 3.6:
```
root@cs-202-15:/lab/work/lego/certificates# cat www.cs-202-15.lab.crt www.cs-202-15.lab.issuer.crt > /lab/tls/server.crt
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-202-15.lab/
curl -sS --cacert /lab/tls/acme-root.crt https://news.cs-202-15.lab/
openssl s_client -connect www.cs-202-15.lab:443 -servername www.cs-202-15.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/14 10:33:08 [notice] 102#102: signal process started
server: cs-202-15 (https)
Host: www.cs-202-15.lab
SNI: www.cs-202-15.lab
protocol: TLSv1.3
server: cs-202-15 (https)
Host: news.cs-202-15.lab
SNI: news.cs-202-15.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.cs-202-15.lab
   i:CN = Pebble Intermediate CA 7d35d9
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 14 10:25:34 2026 GMT; NotAfter: Nov 14 10:25:33 2031 GMT
 1 s:CN = Pebble Intermediate CA 7d35d9
   i:CN = Pebble Root CA 697adc
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 14 08:58:23 2026 GMT; NotAfter: Nov 14 08:58:23 2056 GMT
---
Verify return code: 0 (ok)
```
| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера |`CN = Pebble Intermediate CA 7d35d9` |
| Видавець сертифіката видавця |`CN = Pebble Root CA 697adc` |
| Імена в SAN |`DNS:www.cs-202-15.lab, DNS:news.cs-202-15.lab` |
| Строк дії (від – до) |З `Nov 14 10:25:34 2026 GMT` по `Nov 14 10:25:33 2031 GMT` |
| Тип ключа сервера і алгоритм підпису |Тип ключа: `id-ecPublicKey` (ECDSA 256 біт); алгоритм підпису: `RSA-SHA256` |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 |У кроці 3.5 — 1 сертифікат (тільки серверний); у кроці 3.6 — 2 сертифікати (серверний + проміжний) |

**1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.** 

Перевірку здійснив тестовий ACME-сервер (Pebble) за допомогою процедури валідації HTTP-01, звернувшись до веб-сервера nginx за спеціальною адресою `/.well-known/acme-challenge/...` для кожного з доменів.
Рядки виводу, що це підтверджують: `2026/11/14 10:25:30 [INFO] [www.cs-202-15.lab] The server validated our request
2026/11/14 10:25:34 [INFO] [news.cs-202-15.lab] The server validated our request`. А також записи у логах `nginx` із відповіддю `200` і запитами від `LetsEncrypt-Pebble-VA`, які завантажували файл челенджу з каталогу `/.well-known/acme-challenge/`.

**2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?** 

Перевірка не пройшла (код помилки `21 (unable to verify the first certificate)`), оскільки у файлі `server.crt` на сервері був наявний тільки сам сертифікат сервера. Клієнт знав кореневий сертифікат (`--cacert`), але не знав проміжного сертифіката (`Pebble Intermediate CA`), який підписав сертифікат сервера. Без проміжного сертифіката клієнт не зміг побудувати безперервний ланцюжок довіри (Chain of Trust) від кінцевого сертифіката до кореневого.

**3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?** 

`id-ecPublicKey` (Elliptic Curve Public Key) — це тип відкритого ключа сервера, який використовується для асиметричного шифрування та узгодження ключів (у нашому випадку — еліптичні криві ECDSA).

`RSA-SHA256` — це алгоритм, за допомогою якого проміжний центр сертифікації підписав цей сертифікат (використавши криптографічний алгоритм RSA та хеш-функцію SHA-256).

Вони не суперечать одне одному, оскільки криптографічний алгоритм, яким володіє і використовує сам сервер для встановлення з'єднання (ECDSA), повністю незалежний від алгоритму цифрового підпису, яким удостостовірний центр (CA) завірив цей сертифікат (RSA). 

**4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?** 

Після перезапуску стенда центр сертифікації Pebble заново генерує свої кореневий та проміжний ключі. Через це новий кореневий сертифікат стенда перестане збігатися з тим, яким був підписаний наш сертифікат у `/lab/tls/server.crt`, і перевірка клієнтом завершиться помилкою довіри (`unable to get local issuer certificate` або `self-signed certificate in certificate chain`).
У реальній інфраструктурі це відповідає ситуації, коли центр сертифікації відкликає проміжний сертифікат (компрометація ключа CA), коли закінчується строк дії сертифіката сервера (наприклад, через 90 днів у Let's Encrypt), або коли сервер втрачає приватний ключ після перевстановлення. Для запобігання відмовам у реальних системах налаштовують автоматичне перевипускання сертифікатів (наприклад, через планувальник `cron` чи `systemd` таймери для `lego` / `certbot`).

## Завдання 4. Діагностика несправностей
Вивід кроку 4.1:
```
root@cs-202-15:/lab/work/lego/certificates# for i in 1 2 3 4 5; do
  n=f$i.cs-202-15.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.cs-202-15.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f1.cs-202-15.lab
   i:CN = f1.cs-202-15.lab
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Feb 12 08:49:51 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f2.cs-202-15.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f2.cs-202-15.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 28 08:49:51 2026 GMT; NotAfter: Oct 25 08:49:51 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Nov 13 08:49:51 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f3.cs-202-15.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f3.cs-202-15.lab'
Certificate chain
 0 s:CN = intra.cs-202-15.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Feb 12 08:49:51 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Nov 13 08:49:51 2031 GMT
---
Verify return code: 0 (ok)
=== f4.cs-202-15.lab
server: web, f4
Certificate chain
 0 s:CN = f4.cs-202-15.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Feb 12 08:49:51 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Nov 13 08:49:51 2031 GMT
---
Verify return code: 0 (ok)
=== f5.cs-202-15.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f5.cs-202-15.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 08:49:51 2026 GMT; NotAfter: Feb 12 08:49:51 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```
| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 |`SSL certificate problem: self-signed certificate` (Недовірений підпис) |`18 (self-signed certificate)` |Самопідписаний сертифікат |`i:CN = f1.cs-202-15.lab` (Issuer збігається з Subject) |Замінити сертифікат на підписаний довіреним Центром сертифікації (CA) |
| f2 |`SSL certificate problem: certificate has expired` (Термін вичерпано) |`10 (certificate has expired)` |Прострочений сертифікат |`NotAfter: Oct 25 ... 2026 GMT` (поточна дата пізніша) |Оновити (випустити новий) сертифікат |
| f3 |`SSL: no alternative certificate subject name matches target host name` (Помилка домена) |`0 (ok)` |Невідповідність імені (SAN/CN не збігається з доменом) |`0 s:CN = intra.cs-202-15.lab` (а запитували `f3.cs-202-15.lab`) |Випустити сертифікат із правильним ім'ям у SAN або перейменувати сервер |
| f4 |(відсутня, запит успішний) (Без помилок) |`0 (ok)` |Успішно (справний сервер) |`server: web, f4` / `Verify return code: 0 (ok)` |Нічого (сервер працює коректно) |
| f5 |`SSL certificate problem: unable to get local issuer certificate` (Відсутній видавець) |`21 (unable to verify the first certificate)` |Сервер не надсилає проміжний сертифікат (неповний ланцюжок) |У ланцюжку присутній лише елемент `0`, а проміжний `1` відсутній |Додати проміжний сертифікат видавця до конфігурації сервера (як я робила в завданні 3.6) |

**1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?** 

Для сервера з помилкою `unable to get local issuer certificate` визначено, що це несправність, пов'язана з тим, що сервер не надсилає проміжний сертифікат, тобто ланцюжок є неповним. У виводі `openssl s_client` ці дві несправності відрізняються тим, що у випадку відсутності проміжного сертифіката в блоці `Certificate chain` присутній лише один елемент (сам сертифікат сервера `0`), тоді як клієнт знає корінь, але не може побудувати міст. Якщо ж сертифікат виданий невідомим центром, сервер зазвичай надсилає повний ланцюжок, проте перевірка все одно завершується помилкою невідомого кореня.

**2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.** 

Якщо серед серверів є сервер з невідповідним ім'ям, утиліта `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, оскільки вона перевіряє виключно криптографічну валідність ланцюжка та підписів, не виконуючи суворого зіставлення hostname проти SNI. Натомість `curl` виконує обов'язкову перевірку відповідності доменного імені в полі SAN або CN сертифікатора, і якщо ім'я в сертифікаті не збігається з адресою запиту, `curl` фіксує помилку заради захисту від атак типу Man-in-the-Middle.   

**3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?** 

На запит із ключем `-k (--insecure)` відповіли б усі п'ять серверів (`f1`, `f2`, `f3`, `f4` та `f5`), оскільки цей ключ повністю скасовує перевірку сертифікатів, термінів їхньої дії та довіри до центру сертифікації. Головним наслідком цього є те, що повністю втрачається будь-яка криптографічна гарантія автентичності сервера, і клієнт більше не може захистити себе від перехоплення трафіку зловмисником.

## Завдання 5. Ціна нового з'єднання
**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:
| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` |1 RTT |1 RTT |0 RTT |
| `time_appconnect` |2 RTT |3 RTT |0 RTT |
| `time_starttransfer` |3 RTT |4 RTT |1 RTT |

`time_connect` **(TCP-рукостискання)**: Завжди вимагає 1 RTT для нових з'єднань і 0 RTT для повторного використання вже встановленого каналу.

`time_appconnect` **(Завершення рукостискання TLS)**: Для TLS 1.3 потрібно загалом 2 RTT (1 на TCP + 1 на об'єднаний TLS-рукостисках), а для TLS 1.2 — 3 RTT (1 на TCP + 2 на розширений TLS-рукостисках). При повторному використанні з'єднання додаткове рукостискання не потрібне (0 RTT).

`time_starttransfer` **(Отримання першого байта)**: Додає ще 1 RTT (відправка HTTP-запиту та очікування відповіді від сервера) до часу завершення рукостискання.

Вивід кроків 5.2 і 5.3:
```
root@cs-202-15:/lab/work# F='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total} %{http_version} %{num_connects}\n'
U=https://cdn.cs-202-15.lab/
for i in 1 2 3; do curl -s -o /dev/null --cacert $STAND_CA -w "$F" $U; done
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert $STAND_CA -w "$F" $U; done
0.050906 0.101077 0.129673 0.154987 0.155050 2 1
0.025456 0.050583 0.080028 0.105571 0.105628 2 1
0.025502 0.050646 0.079852 0.105168 0.105224 2 1
0.025402 0.050518 0.104768 0.130090 0.130152 2 1
0.025494 0.050622 0.104778 0.130083 0.130138 2 1
0.025365 0.050494 0.104443 0.129760 0.129828 2 1

root@cs-202-15:/lab/work# curl -s --cacert $STAND_CA -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
0.050799 0.075923 0.104929 0.130240 0.130291 2 1
0.000018 0.000000 0.000000 0.025212 0.025274 2 0
0.000018 0.000000 0.000000 0.025210 0.025252 2 0
```
<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->
| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове |25.5 |50.6 |80.0 |105.6 |105.6 |1 |
| TLS 1.2, нове |25.4 |50.5 |104.8 |130.1 |130.1 |1 |
| 5.3, перший запит |50.8 |75.9 |104.9 |130.2 |130.3 |1 |
| 5.3, другий запит |0.02 |0.0 |0.0 |25.2 |25.3 |0 |
| 5.3, третій запит |0.02 |0.0 |0.0 |25.2 |25.3 |0 |

RTT, визначений вимірюванням, і з якого значення його визначено: 
≈ 25.2 мс. Найточніше значення RTT визначається за часом `starttransfer` для другого та третього запитів у кроці 5.3 (25.21 мс), оскільки з'єднання вже встановлено і відбувається лише передача HTTP-запиту та отримання першого байта відповіді (1 RTT). Також це підтверджується різницею між `time_connect` і `time_namelookup` для нових з'єднань: 50.583 − 25.456 = 25.13 мс (TLS 1.3) та 50.518 − 25.402 = 25.12 мс (TLS 1.2).

**1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?** 

Оскільки `curl` фіксує час наростаючим підсумком від запуску команди (разом із DNS-запитом `namelookup` ≈ 1 RTT), кожен етап відповідає такій кількості RTT:

TLS 1.3, нове з'єднання: `namelookup` ≈ 1 RTT (25.5 мс), `connect` ≈ 2 RTT (50.6 мс), `appconnect` ≈ 3.17 RTT (3 RTT + 4.4 мс = 80.0 мс), `starttransfer` ≈ 4.19 RTT (4 RTT + 4.8 мс = 105.6 мс).

TLS 1.2, нове з'єднання: `namelookup` ≈ 1 RTT (25.4 мс), `connect` ≈ 2 RTT (50.5 мс), `appconnect` ≈ 4.16 RTT (4 RTT + 4.0 мс = 104.8 мс), `starttransfer` ≈ 5.16 RTT (5 RTT + 4.1 мс = 130.1 мс).

Повторне використання (крок 5.3, 2-й і 3-й запити): `starttransfer` = 1.00 RTT (25.2 мс).

Відхилення від цілого числа RTT на етапі `appconnect` (близько 4–4.5 мс) пояснюється часом обчислень на CPU: клієнт і сервер витрачають цей час на криптографічні операції (перевірку ланцюжка сертифікатів, обчислення ключів сесії та генерацію підписів) і внутрішню обробку запиту вебсервером.

**2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?** 

Резолвер (`10.53.0.53`) працює в окремому контейнері мережі, а на вихідному інтерфейсі нашого контейнера стенд додає штучну затримку (`netem`), рівну 1 RTT (≈ 25.2 мс) на кожен мережевий обмін. Тому навіть отримання відповіді з кешу резолвера потребує одного мережевого проходу туди-назад (1 RTT).

Час `time_namelookup` був би приблизно вдвічі більшим (≈ 50.8 мс, що видно у першому запуску 5.2 та 5.3), якби виконувалося два послідовні мережеві запити — наприклад, при зверненні резолвера до авторитетного сервера зони після закінчення TTL запису в кеші або при окремому запиті `AAAA` після `A`.

**3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.** 

Беремо мережевий RTT=25.2 мс (без урахування DNS-пошуку, враховуючи лише витрати на встановлення з'єднання та передачу HTTP-запиту/відповіді: 3 RTT на запит для TLS 1.3, 4 RTT для TLS 1.2 та 1 RTT для повторного запиту):

Нове з'єднання TLS 1.3 для кожного запиту: 

200×3 RTT=600 RTT⟹600×25.2 мс=15120 мс (15.12 с).

Нове з'єднання TLS 1.2 для кожного запиту: 

200×4 RTT=800 RTT⟹800×25.2 мс=20160 мс (20.16 с).

Усі запити через одне з'єднання (TLS 1.3): перший запит потребує 3 RTT, решта 199 запитів — по 1 RTT. Разом: 

3+199×1=202 RTT⟹202×25.2 мс=5090.4 мс (5.09 с).

## Висновки
**Що виявилося несподіваним.** 

Очікувалося, що при використанні HTTPS абсолютно весь трафік між клієнтом і сервером буде повністю зашифрованим і недоступним для перегляду в дампі `tcpdump`. Проте у виводі кроку 1.2 (`grep -a -o 'cdn.cs-202-15.lab' t1-https.txt | wc -l` повернув 1) та кроку 2.3 (`extension_type=server_name(0), length=22`) виявилося, що доменне ім'я сервера передається відкритим текстом у розширенні SNI повідомлення `ClientHello`.

Також несподіваним стало те, що у завданні 4 для сервера `f3.cs-202-15.lab` (де ім'я в сертифікаті `intra.cs-202-15.lab` не збігається з адресою запиту) утиліта `openssl s_client` повернула статус `Verify return code: 0 (ok)`, хоча `curl` заблокував з'єднання з помилкою. Як з'ясувалося, `openssl s_client` за замовчуванням перевіряє лише ланцюжок підписів центру сертифікації, але не звіряє доменне ім'я.

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

У завданні 5 (крок 5.2) під час першого запиту по TLS 1.3 час пошуку доменного імені `time_namelookup` склав `0.050906` с (близько 2 RTT), а у двох наступних спробах зменшився вдвічі — до `0.025456` с (1 RTT). Причиною могло бути закінчення часу життя запису (TTL) у локальному кеші резолвера або паралельний запит IPv6-адреси (`AAAA`), проте без перегляду дампа DNS-трафіку на порту 53 точно підтвердити, чим саме була викликана ця подвійна затримка в першому рядку виводу 5.2, не вдалося.

## Декларація використання ШІ
Використовувалися інструменти ШІ: <так >
<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->
| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 |Gemini 3.6 Flash |Привіт. Буду робити лабу2 з основ побудови інформаційних систем та мереж. Ось методичка до цієї лаби. Розпиши мені кожен етап роботи покроково, що за чим і як робити. Мій id - cs-202-15. | Покроково отримувала виводи за поясненнями від джеміні. Були використані відповіді на питання та таблиці, які джеміні мені генерував за моїми виводами, які я йому надавала.|
|2|Gemini 3.6 Flash|Ось моя робота. Перевір її на помилки чи неточності. А також напиши невеликий висновок простими словами  | Продивилася та перевірила його рекомендації щодо відповідей, висновку та помилок. Зауваження щодо помилок були перевірені і дещо я виправила. Висновки були сформовано нормально, тому я їх залишила. Усі команди в терміналі та вимірювання виконано самостійно.|

