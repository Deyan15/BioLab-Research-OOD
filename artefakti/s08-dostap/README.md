# Сесия 8 – Идентичност и достъп (Keycloak)

Keycloak 26.8.2 (dev режим) на srv (10.90.0.10:8080), realm `biolab`. Контролите са от политиката [03-politika-DOSTAP](../s05-politika/03-politika-DOSTAP.docx) (ULD-01…03, AVT-01, ACC-01, OTD-01, LOG-01) и адресират R-10, R-04 и R-05.

- [test-matrica.docx](test-matrica.docx) – наложените настройки, mover, тестовете T1–T8, находките и SoA статусите

## Наложено

| Контрол | Настройка в Keycloak |
|---|---|
| ULD-01 | Роли и групи `admins`, `analysts`, `staff`; правата идват само от групата |
| ULD-03 | Постоянен именуван админ в realm master; временният `admin` е изтрит |
| AVT-01 | Password policy: Minimum length 15, Not username, Not recently used 3 |
| ACC-01 | Brute force: Lockout temporarily, 5 опита, 15 минути |
| OTD-01 | Поток `browser-mfa-admins`: TOTP задължителен по роля `admins`; Direct access grants: Off |
| LOG-01 | Save events: On, Expiration 90 дни |

## Доказателства от работещата система

| Файл | Какво показва |
|---|---|
| realm-biolab.json | Експорт на realm-а: роли, групи, политики, клиенти (без потребители) |
| sabitiya.txt | `kc-sabitiya biolab 100`: входове и откази с час, потребител, клиент и IP |
| otkaz-parola.txt | T4: парола `kratka` е отказана, `invalidPasswordMinLengthMessage` |
| direct-grant-predi.txt | T8 преди: password grant издава токен с роля `admins` без TOTP |
| direct-grant-sled.txt | T8 след: `unauthorized_client`, Direct access grants са изключени |
| sesii-maria-predi.txt, sesii-maria-sled.txt | T7: една активна сесия преди, `[ ]` след Enabled Off + logout |
| mover-grupi-predi.txt, mover-grupi-sled.txt | Mover: petar.izsledovatel от `/staff` в `/analysts` (смяна, не добавяне) |
| master-users.txt | Потребителите в realm master: само именуваният админ |

## Обратът (T8)

След като TOTP стана задължителен по роля за браузъра, нов администратор все пак получи токен с роля `admins` само с парола през password grant на тестовия клиент. Скриптът сменя само браузър потока, а password grant минава през отделния поток „direct grant“, който иска OTP само от потребители, които вече са го настроили. Поправката е Direct access grants: Off; остава да се проверят всички клиенти, останалите потоци и realm master, чиято админ конзола още няма втори фактор.

## Екранни снимки (поддържащо доказателство)

QR кодовете за TOTP са закрити, защото съдържат тайния ключ.

| Файл | Тест |
|---|---|
| T1-staff-vhod-bez-kod.png | T1: изследовател влиза само с парола |
| T2-hq-admin-totp-setup.png | T2: hq.admin е принуден да настрои TOTP |
| T3-greshen-kod.png | T3: „Invalid authenticator code.“ |
| 4v-nov-admin-bez-totp.png | Находка 4в: нов админ влиза без TOTP преди mfa-po-rolya.sh |
| T5-zaklyuchen-ivan.png | T5: вярната парола е отказана след 5 грешни опита |
| T6-nov-admin-totp-setup.png | T6: нов админ е принуден да настрои TOTP след mfa-po-rolya.sh |
| T7-napusnal-otkaz.png | T7: „Account is disabled, contact your administrator.“ |
| T8-T4-terminal.png | Изходът от mfa-po-rolya.sh, T8 преди и след, T4 |
| sabitiya-grep.png | Файловете в s08-dostap и ключовите събития от журнала |
