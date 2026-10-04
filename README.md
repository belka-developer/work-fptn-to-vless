# fptn-vless

**Форк-надстройка над [FPTN](https://github.com/fptn-project/fptn) (MIT):** подключается к серверам FPTN по обычному токену и отдаёт туннель в локальную сеть как **VLESS-inbound**. Ничего не ставится в систему: ни сетевого адаптера, ни маршрутов, ни изменений DNS.

> English summary: `fptn-vless.exe` is a Windows build of the FPTN client that, instead of creating a Wintun adapter, terminates the tunnel in a userspace lwIP stack and exposes it to the LAN as a plain VLESS inbound (TCP). Phones and other PCs connect with any VLESS client. It adds a one-page web control panel, a tray icon, ping-based multi-server selection and failover. Upstream changes are ~25 lines in 3 files; everything else lives in `src/fptn-vless/`. Details below (in Russian).

![Панель управления](https://raw.githubusercontent.com/belka-developer/media-readme/main/fptn-to-vless/panel-best.png)

## Зачем это нужно

Официальный клиент FPTN создаёт на ПК виртуальный адаптер (Wintun) и меняет маршруты. Это ломает локальные сети поверх виртуальных адаптеров (Radmin VPN, Hamachi и подобные) и заставляет гнать через VPN весь компьютер.

`fptn-vless` работает иначе:

* туннель заканчивается **внутри процесса** (userspace-стек lwIP), система его не видит;
* к туннелю подключаются только те, кому вы дали ссылку `vless://…`: телефон, второй ПК, роутер с VLESS-клиентом;
* остальной трафик ПК идёт как обычно, локальные сети не затрагиваются;
* закрыли программу, и всё осталось как было.

<p align="center"><img src="https://raw.githubusercontent.com/belka-developer/media-readme/main/fptn-to-vless/architecture.svg" alt="Схема" width="900"></p>

## Как это работает

1. **Токен.** Тот же токен доступа, что у официального клиента (`fptnb:…`). Разбор, вход в API и WebSocket-туннель с обходом блокировок (подмена SNI и т. д.) выполняет штатная библиотека FPTN, исходники клиента не переписаны.
2. **Подмена TUN-устройства.** В upstream `TunInterface` на Windows это Wintun. При сборке с `FPTN_USERSPACE_TUN` вместо него подставляется `UserspaceTunDevice`: IP-пакеты библиотеки FPTN попадают в lwIP, работающий в отдельном потоке. `route_manager` в `VpnManager` сделан необязательным (nullptr), поэтому маршруты не трогаются.
3. **VLESS-inbound.** Минимальная реализация: TCP, цели IPv4 или домен, аутентификация по UUID (по умолчанию выводится из токена, можно задать `--uuid`). Каждое VLESS-соединение превращается в TCP-соединение в lwIP, а домены резолвятся внутри туннеля через DNS сервера FPTN (с публичным запасным резолвером).
4. **Выбор сервера и failover.** Программа замеряет отклик всех серверов из токена, подключается к лучшему и раз в 5 минут перемеряет. Переключение с гистерезисом (медленнее лучшего в 1,5 раза **и** минимум на 150 мс, подтверждение повторным замером), новое соединение поднимается до закрытия старого. Ссылка и UUID при смене сервера не меняются.
5. **Самопроверка.** После подключения идёт проверка через туннель: TCP на IP и на имя. Результат и счётчики пакетов видны в панели, так понятно, что именно сломано.

## Веб-панель

Одна страница без прокрутки: слева серверы (сортировка по пингу или по списку), справа настройки. Работает на `http://127.0.0.1:10801/` и принимает команды только с локального компьютера (проверка Host и заголовка против CSRF).

| | |
|---|---|
| **Режим подключения** | «Лучший по пингу» или «Только выбранный» сервер |
| **Если сервер перестал отвечать** | «Перейти на другой» или «Ничего не делать» |
| **Порт VLESS** | любой свой; если занят, остаётся прежний |
| **Брандмауэр** | кнопка «Открыть порт в брандмауэре» (запрос UAC) и индикатор состояния |
| **Ссылка** | `vless://…` с адресом ПК в локальной сети, кнопка «Копировать» |
| **Токен** | ввод и смена прямо в панели, хранится зашифрованным (DPAPI) только для вашей учётной записи Windows |

<p align="center">
<img src="https://raw.githubusercontent.com/belka-developer/media-readme/main/fptn-to-vless/panel-selected.png" width="48%" alt="Режим «Только выбранный»">
<img src="https://raw.githubusercontent.com/belka-developer/media-readme/main/fptn-to-vless/panel-dark.png" width="48%" alt="Тёмная тема">
</p>
<p align="center"><img src="https://raw.githubusercontent.com/belka-developer/media-readme/main/fptn-to-vless/panel-token.png" width="60%" alt="Ввод токена"></p>

## Использование

1. Запустите `fptn-vless.exe`. Он создаст ярлык на рабочем столе, откроет панель и поселится в трее.
2. Вставьте токен, нажмите «Загрузить серверы», затем подключитесь к лучшему или выберите сервер.
3. Нажмите «Копировать» у ссылки и вставьте её в VLESS-клиент (Incy, v2rayN, Streisand и т. п.) на телефоне или другом ПК в той же сети.
4. Если устройства не подключаются, нажмите «Открыть порт в брандмауэре».
5. Дальше достаточно ярлыка: он открывает панель, а если программа не запущена, запускает её и сам подключается к сохранённому серверу.

Параметры командной строки нужны только для особых случаев: `--access-token`, `--token-file`, `--server`, `--port`, `--dashboard-port`, `--link-host`, `--no-failover`, `--scan-interval`, `--no-tray`, `--no-shortcut`, `--no-browser`, `--verbose`. Подробности и таблица параметров лежат в `src/fptn-vless/README.md` внутри патча.

## Что изменено относительно upstream

Работает как **патч поверх FPTN** (закреплён коммит `c164fa9`), сам upstream не форкнут и не изменяется:

* `src/common/network/net_interface.h`: под `FPTN_USERSPACE_TUN` вместо Wintun/Linux TUN используется `UserspaceTunDevice`;
* `src/fptn-client/vpn/vpn_manager.cpp`: `route_manager` необязателен;
* `CMakeLists.txt`: подключена цель `fptn-vless`;
* всё остальное добавлено новым каталогом `src/fptn-vless/` (lwIP-обвязка, VLESS, failover, панель, трей, настройки).

Репозиторий содержит только `fptn-vless.patch` и workflow GitHub Actions: он забирает upstream на нужном коммите, применяет патч и собирает `fptn-vless.exe` (Windows, Conan, MSVC 2022).

### Сборка

```
Actions → «Build fptn-vless.exe» → Run workflow → артефакт fptn-vless-windows-x64
```

Локально: клонировать `fptn-project/fptn` на коммит `c164fa9`, выполнить `git apply --whitespace=nowarn fptn-vless.patch`, собрать цель `fptn-vless` с `-DFPTN_BUILD_VLESS=ON` по штатной инструкции FPTN для Windows.

## Ограничения

* Только TCP. VLESS-UDP (QUIC/HTTP3) не поддержан, приложения откатываются на TCP.
* Цели только IPv4 или доменные имена.
* VLESS без TLS: только для доверенной домашней сети. Не пробрасывайте порт в интернет.
* Целевая платформа: Windows. На Linux собираются только тесты.

## Статус тестирования

Автотесты (Linux, подставные серверы): VLESS → lwIP → IP-пакеты → второй lwIP (эхо, 4 МиБ в обе стороны, 8 параллельных потоков, неверный UUID, таймаут), логика failover, защита страницы, жизненный цикл приложения и настройки.

**Ещё не проверено на практике:** работа с реальным сервером FPTN под нагрузкой, трей, ярлык и кнопка брандмауэра (UAC) на разных версиях Windows, совместимость с конкретными VLESS-клиентами. Сообщения об ошибках приветствуются; в issue приложите строку `Tunnel check: …` из журнала панели.

## Лицензия и благодарности

Основа: [FPTN](https://github.com/fptn-project/fptn) (MIT), стек [lwIP](https://savannah.nongnu.org/projects/lwip/) (BSD-3). Токены доступа и серверы принадлежат их владельцам; проект не обходит ограничения доступа к серверам.
