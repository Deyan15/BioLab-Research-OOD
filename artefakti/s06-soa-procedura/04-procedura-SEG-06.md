# Процедура: SSH достъп до Transfer Station само с ключ и само от администраторската машина

МИС, ТУЕС · 04-procedura · фирма 10, BioLab Research OOD

| Поле | Стойност |
| --- | --- |
| Реализира контрол | SEG-06: Защитен администраторски достъп до Transfer Station |
| Annex A | 8.5 Сигурна автентикация; 8.2 Привилегировани права за достъп; 8.22 Разделяне на мрежите; 5.37 Документирани оперативни процедури |
| R-ID | R-05: отгатване на SSH паролата на Transfer Station (A-03) и използването ѝ като мост към лабораторията (риск 15, висок) |
| Изпълнява се от | HQ администратор |
| Честота | При първоначална настройка; при всяка нова Transfer Station; до 24 ч след смяна на HQ администратора (нов ключ и нов адрес). Проверка на действащите стойности (стъпки 9 и 10): веднъж месечно до 5-о число, от Служител по МИС |
| Версия, дата | 0.1, 12.11.2026 |

## Предусловия

| Какво | Стойност |
| --- | --- |
| Достъп | Акаунт `student` на srv (A-03, 10.90.0.10) с права за `sudo`; акаунт `student` на gw (A-01, 10.90.0.1), който в лабораторията е администраторската машина на HQ администратора |
| Инструменти | `openssh-server` на srv (услуга `ssh`); `openssh-client` на gw; ключ `ed25519` на `student` от gw, копиран в `/home/student/.ssh/authorized_keys` на srv (на gw: `ssh-keygen -t ed25519`, после `ssh-copy-id student@10.90.0.10`) |
| Снапшот преди започване | `pre-h06` на srv (VirtualBox: Machine → Take Snapshot) |
| Първа сесия | Конзолата на srv във VirtualBox (локален вход, не зависи от SSH). Всички стъпки `srv#` се изпълняват в нея, в директория `/home/student/h06`. Всяка команда се въвежда на един ред; Enter се натиска само в края ѝ |
| Втора сесия | SSH от gw към srv: `ssh student@10.90.0.10`. Отворена е преди стъпка 1 и остава отворена до края. Стъпките `ssh-сесия$` се изпълняват в нея |

## Стъпки

Означения: `srv#` е конзолата на srv; `ssh-сесия$` е втората сесия (SSH от gw към srv); `gw$` е нов терминал на gw, без SSH към srv. Ако резултатът е различен от очаквания: спри и мини към „При отклонение“.

| # | Действие | Команда | Очакван резултат |
| --- | --- | --- | --- |
| 1 | Създай работна директория и запиши състоянието преди (5 отделни команди) | `srv# mkdir -p ~/h06 && cd ~/h06`, после `srv# date > predi.txt`, после `srv# hostname >> predi.txt`, после `srv# sudo sshd -T \| grep -Ei 'permitrootlogin\|passwordauthentication\|kbdinteractive\|allowusers' >> predi.txt`, после `srv# cat predi.txt` | Дата, `srv` и текущите стойности (без ред `allowusers`) |
| 2 | Запиши адреса на администраторската машина, както го вижда srv | `ssh-сесия$ echo $SSH_CLIENT \| cut -d' ' -f1 \| tee ~/h06/admin-ip.txt` | `10.90.0.1` |
| 3 | Провери, че входът с ключ работи, без парола | `gw$ ssh -o PasswordAuthentication=no -o BatchMode=yes student@10.90.0.10 true; echo $?` | Отпечатва `0`, без да пита за парола. Ако не е `0`: спри |
| 4 | Провери, че файловете от `sshd_config.d` се четат първи | `srv# grep -n '^Include' /etc/ssh/sshd_config` | `Include /etc/ssh/sshd_config.d/*.conf` е преди всички други настройки във файла |
| 5 | Създай отделен файл с настройката (5 отделни команди; първата е с `tee`, следващите с `tee -a`) | `srv# echo 'PermitRootLogin no' \| sudo tee /etc/ssh/sshd_config.d/10-seg06.conf`, после `srv# echo 'PasswordAuthentication no' \| sudo tee -a /etc/ssh/sshd_config.d/10-seg06.conf`, после `srv# echo 'KbdInteractiveAuthentication no' \| sudo tee -a /etc/ssh/sshd_config.d/10-seg06.conf`, после `srv# echo "AllowUsers student@$(cat ~/h06/admin-ip.txt)" \| sudo tee -a /etc/ssh/sshd_config.d/10-seg06.conf`, после `srv# cat /etc/ssh/sshd_config.d/10-seg06.conf` | Файлът има точно 4 реда; последният е `AllowUsers student@10.90.0.1` |
| 6 | Провери синтаксиса | `srv# sudo sshd -t; echo $?` | Няма съобщение за грешка; отпечатва `0` |
| 7 | Приложи, без да прекъсваш връзките | `srv# sudo systemctl reload ssh && systemctl is-active ssh` | `active`; втората сесия е жива (Enter в нея → нов промпт) |
| 8 | Положителен тест с нова връзка от администраторската машина | `gw$ ssh -o PasswordAuthentication=no -o BatchMode=yes student@10.90.0.10 true; echo $?` | `0` |
| 9 | Запиши състоянието след (от работещата услуга) | `srv# date > sled.txt`, после `srv# hostname >> sled.txt`, после `srv# sudo sshd -T \| grep -Ei 'permitrootlogin\|passwordauthentication\|kbdinteractive\|allowusers' >> sled.txt`, после `srv# cat sled.txt` | `permitrootlogin no`, `passwordauthentication no`, `kbdinteractiveauthentication no`, `allowusers student@10.90.0.1` |
| 10 | Отрицателни тестове (виж „Проверка“) | командите от „Проверка → Отрицателна“ | Трите опита са отказани; записани в `otkaz.txt` (gw) и `otkaz-lokalno.txt` (srv) |
| 11 | Запиши следата в журнала | `srv# sudo journalctl -u ssh --since "1 hour ago" --no-pager > zhurnal.txt`, после `srv# grep -E 'Reload\|not allowed\|Accepted publickey' zhurnal.txt` | Ред за `Reload`, ред `Accepted publickey for student from 10.90.0.1` и ред `User student from 127.0.0.1 not allowed because not listed in AllowUsers` |

## Проверка

**Положителна (разрешеното работи):**

Стъпка 8: `gw$ ssh -o PasswordAuthentication=no -o BatchMode=yes student@10.90.0.10 true; echo $?` → `0`. Нова връзка с ключ от администраторската машина влиза без парола.

**Отрицателна (забраненото се отказва):**

Всеки опит е нова връзка, не през вече отворената втора сесия.

1. Вход с парола от администраторската машина: `gw$ ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password student@10.90.0.10 2>&1 | tee otkaz.txt` → `student@10.90.0.10: Permission denied (publickey).`
2. Вход като root: `gw$ ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password root@10.90.0.10 2>&1 | tee -a otkaz.txt` → `root@10.90.0.10: Permission denied (publickey).`
3. Вход от адрес, различен от администраторската машина (в лабораторията няма A-05, затова адресът 127.0.0.1 на самия srv замества „всяка друга машина“), в `~/h06`: `srv# ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new student@127.0.0.1 2>&1 | tee otkaz-lokalno.txt` → `Permission denied (publickey).`; после `srv# sudo journalctl -u ssh --since "5 min ago" --no-pager | grep 'not allowed' | tee -a otkaz-lokalno.txt` → ред `User student from 127.0.0.1 not allowed because not listed in AllowUsers`

`otkaz.txt` се създава на gw, `otkaz-lokalno.txt` на srv. Ако някой опит поиска парола или влезе: контролът не е наложен, спри.

## При отклонение

Ако стъпка 2, 3, 4, 6, 7 или 8 не даде очаквания резултат: спри и не изпълнявай следващата стъпка. Ако файлът `10-seg06.conf` вече е създаден, изпълни „Възстановяване“ от конзолата на srv. Запиши номера на стъпката и точното съобщение в `~/h06/otklonenie.txt` и ескалирай към Служител по МИС до 4 работни часа. Ако стъпка 9, 10 или 11 даде различен резултат, контролът не работи: изпълни „Възстановяване“ и ескалирай по същия начин.

## Възстановяване (Rollback)

От конзолата на srv (първата сесия), която не зависи от SSH:

```
sudo rm /etc/ssh/sshd_config.d/10-seg06.conf
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|allowusers)'
```

Очаквано: стойностите са като в `predi.txt`, няма ред `allowusers`. Ако srv не отговаря и в конзолата: върни снапшота `pre-h06` (VirtualBox: Machine → Snapshots → pre-h06 → Restore).

## Запис

`predi.txt`, `admin-ip.txt`, `sled.txt`, `otkaz-lokalno.txt`, `zhurnal.txt` (от srv, в `~/h06`) и `otkaz.txt` (от gw) се копират в хранилището на проекта, папка `artefakti/s06-soa-procedura/`, и се пазят 12 месеца, както изисква SEG-06. Всяка месечна проверка добавя `sled-<ГГГГ-ММ>.txt` (стъпка 9) в същата папка.

## Проверка от съученик

| Поле | Стойност |
| --- | --- |
| Изпълнил, дата | Денис Димитров, 11.11.2026 |
| Засякъл на стъпка | не |
| Неясно | няма |
| Отрицателният тест даде | Отказ, както е описано: `Permission denied (publickey)` |
| Поправено след проверката | Не е било нужно, процедурата е изпълнена без грешки |
