# BRC v3 — KolibriOS

Клиент для браузерного агента BRC, написанный на FASM для KolibriOS.  
Адаптивный UI: все элементы пересчитываются при каждом resize окна.

## Сборка

```
/sys/Develop/Fasm /hd0/1/brc_kolibri.asm /hd0/1/brc_kolibri
```

## Архитектура UI

```
┌──────────────────────────────────────┐
│  BRC v3 for KolibriOS         [─][×] │  заголовок скина
├──────────────────────────────────────┤
│  Request:                            │
│  [поле ввода ── dynEW ──] [Send ]    │  draw_edit + кнопки dynBX
│                           [Clear]   │
│  Response:                           │
│  ┌──────── dynOW ─────────────────┐  │
│  │                                │  │  draw_out, высота dynOH
│  └────────────────────────────────┘  │
├──────────────────────────────────────┤
│  Ready. Type and press Enter...      │  draw_status, Y = dynSY
└──────────────────────────────────────┘
```

## Как работает адаптация

1. Ядро KolibriOS посылает **Event 1** (redraw) при любом изменении окна
2. `get_wnd_size` вызывает **fn9** → читает `client_box.width` (offset 42) и `client_box.height` (offset 46)
3. Из этих значений вычисляются все динамические координаты:

| Переменная | Формула | Назначение |
|------------|---------|------------|
| `dynEW` | `wW - EX*2 - BTN_W - 16` | ширина поля ввода |
| `dynOW` | `wW - OX*2` | ширина поля вывода |
| `dynOH` | `wH - OY - 40` | высота поля вывода |
| `dynSY` | `wH - 22` | Y статус-бара (прилипает к низу) |
| `dynBX` | `wW - BTN_W - 8` | X кнопок (прилипают к правому краю) |

4. `wnd_draw` перерисовывает всё с нуля с новыми координатами

## Файлы

- **[header.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/header.asm)** — Заголовок MENUET01, константы окна, цвета, отступы  
- **[entry.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/entry.asm)** — Точка входа: cfg_load → get_wnd_size → wnd_draw  
- **[evtloop.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/evtloop.asm)** — Главный цикл событий: redraw / key / button / resize / timer  
- **[get_wnd_size.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/get_wnd_size.asm)** — Читает client_box.width/height через fn9 (offset 42/46), пересчитывает dynEW/OW/OH/SY/BX  
- **[check_resize.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/check_resize.asm)** — Перерисовывает окно только если размер реально изменился  
- **[get_time.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/get_time.asm)** — Возвращает системное время в сотых долях секунды (fn26 sf9)  
- **[txt_draw.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/txt_draw.asm)** — Вывод строки текста: вычисляет длину, вызывает fn4  
- **[cp866_to_utf8.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/cp866_to_utf8.asm)** — Конвертирует строку CP866 → UTF-8 для отправки на сервер  
- **[parse_hex4.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/parse_hex4.asm)** — Разбирает 4 hex-символа в число (для \uXXXX в JSON)  
- **[unicode_to_cp866.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/unicode_to_cp866.asm)** — Конвертирует Unicode codepoint → CP866 байт (кириллица)  
- **[wnd_draw.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/wnd_draw.asm)** — Полная перерисовка окна: fn12 begin/end, создание окна, метки, кнопки, поля  
- **[draw_edit.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/draw_edit.asm)** — Поле ввода: фон, рамка (fn38), текст, курсор, placeholder  
- **[draw_out.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/draw_out.asm)** — Область вывода ответа: фон, рамка, постраничный вывод строк  
- **[draw_status.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/draw_status.asm)** — Строка статуса внизу окна — прилипает к dynSY, показывает Ready/Sending/Done  
- **[key_hdl.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/key_hdl.asm)** — Обработка клавиатуры: Enter → do_send, BS, ESC, обычные символы  
- **[do_send.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/do_send.asm)** — Отправка запроса: cp866→utf8→base64, 4 вызова brc_post, brc_read, parse_resp  
- **[parse_resp.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/parse_resp.asm)** — Разбор JSON-ответа: eval_result → tmpBuf, поиск ,"t":" → outBuf с конвертацией \n  
- **[brc_post.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/brc_post.asm)** — Сборка JSON и HTTP POST для команд BRC (count/hover/confirm)  
- **[brc_fill.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/brc_fill.asm)** — HTTP POST команды fill — отправляет base64-закодированный текст  
- **[brc_read.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/brc_read.asm)** — HTTP POST команды read — читает ответ из браузера через JS  
- **[http_post.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/http_post.asm)** — Низкоуровневый TCP: socket → connect → send → recv → close (fn75)  
- **[build_req.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/build_req.asm)** — Собирает HTTP-запрос: POST заголовки + Content-Length + тело  
- **[parse_ip.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/parse_ip.asm)** — Разбирает IP-адрес из строки в 4 байта для sockAddr  
- **[do_b64_buf.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/do_b64_buf.asm)** — Base64-кодирование буфера (стандартный алфавит, дополнение =)  
- **[cfg_load.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/cfg_load.asm)** — Загружает brc.ini через fn70, при ошибке использует встроенные defaults  
- **[ini_parse.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/ini_parse.asm)** — Разбирает INI-файл: api_key, browser_id, host_ip, host, path, sleep  
- **[key_cmp.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/key_cmp.asm)** — Сравнивает ключ INI-строки с образцом (без нуль-терминатора в ключе)  
- **[copy_val.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/copy_val.asm)** — Копирует значение INI (до CR/LF) в целевой буфер  
- **[parse_dec.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/parse_dec.asm)** — Разбирает десятичное число из строки → eax  
- **[write_log.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/write_log.asm)** — Дописывает запрос+ответ в /hd0/1/brc.log через fn70 sf3/sf5  
- **[strings.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/strings.asm)** — Строковые утилиты: s_cat, s_cpy, s_len, wrt_dec, s_str  
- **[data.asm](https://git.kolibrios.org/dictovod/BRC/src/branch/main/data.asm)** — Секция данных: строки, буферы, переменные состояния, конфиг  

## Конфиг brc.ini

```ini
[brc]
api_key=ваш_ключ
browser_id=ваш_id
host_ip=31.31.197.18
host=lp85d.ru
path=/wp-json/brc/v1/command
sleep=1000
```

Файл читается при запуске через **fn70**. При отсутствии используются встроенные defaults.

## Поток запроса

```
Ввод текста (CP866)
  → cp866_to_utf8
  → do_b64_buf (Base64)
  → brc_post(__brc_count_responses__)
  → brc_post(__brc_hover_last_user_message__)
  → brc_fill(__brc_fill_edit_field__:<base64>)
  → brc_post(__brc_confirm_edit__)
  → sleep(cfgSleep ms)
  → brc_read (JS читает ответ из DOM)
  → parse_resp (eval_result → ,"t":" → outBuf)
  → draw_out (вывод на экран)
```
<img width="329" height="393" alt="image" src="https://github.com/user-attachments/assets/3fd4a1d8-bb36-4d99-a8e2-0ffc60c8568a" />
