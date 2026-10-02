# fptn-vless-build

Патч к [fptn-project/fptn](https://github.com/fptn-project/fptn) и workflow,
который собирает `fptn-vless.exe`. Исходники FPTN здесь не хранятся: они
скачиваются из официального репозитория при каждой сборке.

Сборка: **Actions → Build fptn-vless.exe → Run workflow**. Готовый exe появится
в артефактах запуска (первый запуск 30-60 минут, дальше быстрее из кэша).
