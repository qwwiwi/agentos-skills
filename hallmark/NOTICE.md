# Hallmark — источник и лицензия

- **Источник:** https://github.com/Nutlope/hallmark (каталог `skills/hallmark/`)
- **Коммит:** `de4999e0cd7a7a41df0b4e6903fa02fe9f1492a8` от 2026-11-04
- **Версия скилла:** 2.0.0
- **Лицензия:** MIT (файл `LICENSE` рядом)
- **Авторы:** Together AI (Nutlope)

Содержимое каталога скопировано без изменений, кроме этого файла `NOTICE.md`.

## Как обновить из upstream

```bash
git clone --depth 1 https://github.com/Nutlope/hallmark /tmp/hallmark-src
rm -rf hallmark && cp -R /tmp/hallmark-src/skills/hallmark hallmark
# вернуть NOTICE.md, обновить коммит и дату в нём
zip -qr hallmark.skill hallmark
```

Скрипты из upstream перед запуском просматривать глазами: это чужой код.
