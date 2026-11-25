# Сесия 6 – SoA и процедура

- [05-soa.xlsx](05-soa.xlsx) – Statement of Applicability: всичките 93 контрола от Annex A с решение и обосновка (R-ID, ред от матрицата на проследимост)
- [04-procedura-SEG-06.md](04-procedura-SEG-06.md) – процедура за контрол SEG-06: SSH до Transfer Station (srv, A-03) само с ключ и само от администраторската машина (R-05; Annex A 8.5, 8.2, 8.22)

Доказателства от работещата система:

| Файл | Какво показва |
|---|---|
| predi.txt | Състоянието преди: `permitrootlogin without-password`, `passwordauthentication yes` |
| admin-ip.txt | Адресът на администраторската машина, както го вижда srv (10.90.0.1) |
| sled.txt | Състоянието след, от `sshd -T`: `no`, `no`, `no`, `allowusers student@10.90.0.1` |
| otkaz.txt | Отрицателни тестове от gw: парола за student и за root → `Permission denied (publickey)` |
| otkaz-lokalno.txt | Вход от адрес, различен от админ машината (127.0.0.1) → отказ от AllowUsers |
| zhurnal.txt | `journalctl -u ssh`: reload, приет ключ от 10.90.0.1, откази |
| srv-predi-sled-zhurnal.png | Екранна снимка от конзолата на srv: преди, настройката, `sshd -t`, reload, след, отказ от 127.0.0.1, журнал (поддържащо доказателство) |
| gw-otkaz.png | Екранна снимка от gw: двата отрицателни теста (поддържащо доказателство) |

Проверка от съученик: Денис Димитров, 11.11.2026, без забележки.
