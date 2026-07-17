# ESPHome USB/IP Сервер для принтера

**Форк:** [tiehfood/esphome-usbip-printer](https://github.com/tiehfood/esphome-usbip-printer)

# Что переделано
- Добавлен текстовый сенсор `wifi_info` (`ip_address`, `ssid`, `mac_address`) в конфигурацию ESPHome для вывода сетевой информации на веб-страницу.
- Добавлен сенсор `wifi_signal` в конфигурацию ESPHome для мониторинга уровня Wi-Fi сигнала.
- Подтверждена работоспособность OTA-прошивки и чтения логов по воздуху по IP-адресу `192.168.1.206` (или по mDNS имени `usbip-server.local`).
- Добавлен C++ функционал в компонент `usbip` для получения статуса USB-принтера и сетевого клиента.
- Добавлены новые шаблонные текстовые датчики `USB Device Status` и `USBIP Client Status` в веб-интерфейсе.
- Удален неиспользуемый файл прошивки `sihpP1102.dl` из конфигурации для принтера HP LaserJet 1010, что сократило размер бинарника.
- Исправлена ошибка отсутствия фильтра `unique` для текстовых сенсоров: фильтрация дубликатов логов реализована непосредственно внутри C++ лямбда-кода конфигурации с возвратом пустого optional.

## Скачать проект на комп с которого будет шиться esp32 (нужно чтоб на компе был установлен git и python3) 
```bash
git clone https://git.6387.ru/fanhoi/esphome-usbip-printer.git
cd esphome-usbip-printer
```
## Установить окружение и esphome
```bash
python3 -m venv venv                                              
source venv/bin/activate
pip install esphome
```
 
## Проверка версии ESPHome
```bash
esphome version
```
## Перед прошивкой в usbip_config.yaml нужно прописать свой wifi
```yaml
wifi:
  ssid: "Your WiFi SSID"
  password: "Your WiFi Password"
```

## Прошивка по usb (usbmodem5B901303871 - порт юсб можно посмотреть командой `ls /dev/cu.*`)
```bash
esphome run usbip_config.yaml --device /dev/cu.usbmodem5B901303871
```

## Прошивка по wifi (после вервой прошивки по юсб, можно повторно шить уже по вайфай)
```bash
venv/bin/esphome run usbip_config.yaml --device usbip-server.local
```

## ручное приаттачивание принтера на самом хосте ProxmoxVE
```bash
usbip attach -r 192.168.1.206 -b 1-1
```
## CUPS в моем примере развернут в lxc контейнере ubuntu 24.04
установку cups рассписывать не буду, так как вариантов много, и в интернете их навалом)
приведу только полезные команды:
```bash
apt install -y cups printer-driver-foo2zjs # установка cups и драйвера для принтера
systemctl enable cups 
systemctl start cups 
nano /etc/cups/cupsd.conf # файл конфигурации CUPS
lpstat -v # проверка подключенных принтеров
lpinfo -v # просмотр драйверов
```
## веб интерфейс CUPS
```
https://<IP_КОНТЕЙНЕРА>:631
```
в веб интерфейсе cups нужно найти и добавить принтер
у меня это `hp:/usb/hp_LaserJet_1010?serial=00CNFD934403`
модель `HP Laserjet 1010, hpcups 3.23.12 (en)`

## В конфигурации контейнера lxc нужно добавить, проброс с хоста, на файлы ipusb чтоб CUPS видел принтер:
```bash
nano /etc/pve/lxc/<CTID>.conf
```
добавить в конце файла:
```bash
unprivileged: 0                   # разрешаем привилегии
lxc.cgroup2.devices.allow: c 180:* rwm   # разрешаем доступ к USB принтеру 
lxc.mount.entry: /dev/usb/lp0 dev/usb/lp0 none bind,optional,create=file
lxc.cgroup2.devices.allow: c 189:* rwm   # разрешаем доступ к USB шине
lxc.mount.entry: /dev/bus/usb dev/bus/usb none bind,optional,create=dir
```

# Автоматическое переподключение после включения принтера
## Создаем скрипт:
```bash
nano /usr/local/bin/usbip-printer-watch.sh
```

## Содержимое файла:
```bash
#!/bin/bash
while true; do
    if ping -c1 -W1 192.168.1.206 >/dev/null 2>&1; then
        if ! usbip port | grep -q "Port in Use"; then
            usbip attach -r 192.168.1.206 -b 1-1 >/dev/null 2>&1
        fi
    fi
    sleep 30
done
```

## Делаем файл исполняемым:
```bash
chmod +x /usr/local/bin/usbip-printer-watch.sh
```

## Создаем systemd сервис
```bash
nano /etc/systemd/system/usbip-printer.service
```

## Содержимое:
```ini
[Unit]
Description=USBIP Printer Auto Attach
After=network-online.target
Wants=network-online.target
[Service]
ExecStart=/usr/local/bin/usbip-printer-watch.sh
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
```

## Активируем сервис:
```bash
systemctl daemon-reload
systemctl enable usbip-printer.service
systemctl start usbip-printer.service
```

## Проверка состояния:
```bash
systemctl status usbip-printer.service
```
## Проверка подключения:
```bash
usbip port
```
## Просмотр логов:
```bash
journalctl -u usbip-printer.service -f
```




## **Оригинальная документация:** [tiehfood/esphome-usbip-printer](https://github.com/tiehfood/esphome-usbip-printer)

USB/IP сервер, работающий на плате Waveshare ESP32-S3-ETH в качестве компонента ESPHome. Он связывает локально подключенное USB-устройство с удаленной Linux-машиной через Ethernet, используя стандартный протокол USB/IP.
```
Linux Клиент                  ESP32-S3-ETH               USB Устройство
 ──────────────                ────────────               ──────────────
 usbip attach ──► TCP:3240 ──► USB/IP сервер ──► USB ──► Принтер
 CUPS / lp    ◄── USB/IP   ◄── компонент     ◄── USB ◄── Клавиатура, ...
```
Linux-хост видит USB-устройство так, как будто оно подключено напрямую. Все USB-операции (управляющие передачи, массовые IN/OUT, дескрипторы) прозрачно перенаправляются.

Реализовано с помощью ИИ. Это POC (доказательство концепции) и работает с моим принтером HP LaserJet P1102w.

## Возможности

- Стандартный протокол USB/IP v1.1.1 — работает с утилитами usbip, поставляемыми с ядром Linux
- Обнаружение горячего подключения USB-устройств (подключение и отключение во время работы)
- Автоматическая загрузка прошивки HP Smart Install для принтера HP LaserJet P1102w
- Обход аппаратной ошибки DWC2 bulk OUT DMA для ESP32-S3

## Предварительные требования

- Оборудование: Waveshare ESP32-S3-ETH (или любая плата на ESP32-S3 с Ethernet и выведенными контактами USB-host)
- Фреймворк: ESPHome 2025.12+ с бэкендом ESP-IDF
- Linux-клиент: утилиты usbip (пакет linux-tools-common в Ubuntu/Debian)

## С чего начать

### 1. Настройка виртуального окружения Python

```bash
python3 -m venv virtenv
source virtenv/bin/activate
pip install esphome
```

### 2. Сборка и прошивка

```bash
source virtenv/bin/activate
esphome compile usbip_config.yaml
esphome upload usbip_config.yaml
```

### 3. Подключение из Linux

```bash
# Discover
usbip list -r <esp32-ip>

# Attach
sudo usbip attach -r <esp32-ip> -b 1-1

# Verify
lsusb

# Detach
sudo usbip detach -p 0
```

После подключения устройство появляется как нативный USB-устройство. Используйте его с CUPS, lp или любым стандартным драйвером.

## Конфигурация

### Параметры компонента

```yaml
usbip:
  port: 3240                        # optional, default 3240
  printer_firmware: sihpP1102.dl    # optional, HP P1102w only
```

| Параметр | Обязательный | Default | Описание |
|-----------|----------|---------|-------------|
| `port` | Нет | `3240` | TCP-порт для USB/IP-сервера. 3240 — стандартный порт USB/IP для Linux. |
| `printer_firmware` | Нет | — | Путь к блобу прошивки принтера HP (относительно файла конфигурации). Если указано, прошивка встраивается во флеш во время компиляции и автоматически загружается при каждом подключении принтера. |

### Требуемые опции sdkconfig ESP-IDF

Должны быть установлены в `esp32.framework.sdkconfig_options`:

| Опция | Значение | Причина |
|--------|-------|--------|
| `CONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE` | `"1024"` | Принтеры возвращают большие дескрипторы и строки Device ID (до ~1 КБ). Значение по умолчанию в ESP-IDF 256 байт приводит к усеченным ответам. |
| `CONFIG_USB_HOST_HW_BUFFER_BIAS_BALANCED` | `y` | Балансирует внутренний FIFO DWC2 между конечными точками IN и OUT. Смещение по умолчанию в сторону периодических конечных точек вызывает нехватку ресурсов для массовых передач. |
| `CONFIG_USB_HOST_ENABLE_ENUM_FILTER_CALLBACK` | `y` | Позволяет компоненту принимать все USB-устройства и контролировать, какая конфигурация выбирается во время перечисления. |
| `CONFIG_SPIRAM` | `n` | Механизм DMA DWC2 требует внутренней SRAM. Буферы на основе PSRAM вызывают сбои DMA или конфликты шины. |

### Обязательные build flags

```yaml
esphome:
  platformio_options:
    build_flags:
      - -DCONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE=1024
      - -DCONFIG_USB_HOST_HW_BUFFER_BIAS_BALANCED=1
```

Эти значения дублируют параметры sdkconfig в виде макросов препроцессора. Некоторые заголовочные файлы ESP-IDF проверяют их через #ifdef, поэтому требуются как запись в sdkconfig, так и флаг сборки.

### Скорость логгирования

```yaml
logger:
  baud_rate: 0
```

**Обязательно.** USB-порты ESP32-S3 (GPIO19/20) используются для JTAG/serial и режима USB Host. `baud_rate: 0` освобождает их для USB Host. Вывод логов доступен через веб-сервер ESPHome или API Home Assistant.

### Полный пример

См. [`usbip_config.yaml`](usbip_config.yaml) для полной рабочей конфигурации, включающей Ethernet, веб-сервер, OTA и диагностику.


## Технические детали

### Протокол USB/IP

Компонент реализует протокол USB/IP версии 0x0111 через одно TCP-соединение:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `OP_REQ_DEVLIST` / `OP_REP_DEVLIST` | Клиент ← → Сервер | Список подключенных USB-устройств с дескрипторами |
| `OP_REQ_IMPORT` / `OP_REP_IMPORT` | Клиент ← → Сервер | Присоединить устройство; соединение переключается в режим URB |
| `USBIP_CMD_SUBMIT` / `USBIP_RET_SUBMIT` | Клиент ← → Сервер | Пересылка блоков запросов USB (control + bulk transfers) |
| `USBIP_CMD_UNLINK` / `USBIP_RET_UNLINK` | Клиент ← → Сервер | Отмена ожидающих передач |

### ESP32-S3 DWC2 Bulk OUT Bug

ESP32-S3 использует контроллер USB Synopsys DWC2 в режиме Scatter-Gather DMA. Этот режим имеет аппаратный сбой: **bulk OUT передачи, требующие нескольких дескрипторов DMA, замораживают контроллер**. Любая передача OUT больше максимального размера пакета (64 байта для Full-Speed) вызывает заморозку.

**Обходной путь:** Данные bulk OUT разбиваются на отдельные части размером 64 байта MPS, каждая из которых передается как отдельная передача через ESP-IDF API. Передачи с одним дескриптором позволяют избежать ошибки. ESP-IDF отслеживает переключение DATA0/DATA1 внутренне.

Прямой режим Buffer DMA  fallback (`USE_DIRECT_BULK_OUT=1`), который обходит ESP-IDF, включен в исходный код, но отключен по умолчанию.

### Выравнивание Bulk IN

ESP-IDF отклоняет bulk IN передачи, размер которых не кратен максимальному размеру пакета конечной точки (`ESP_ERR_INVALID_ARG`). Компонент округляет все выделения буферов IN до следующей границы в 64 байта.

### Горячее подключение устройств

1. **Connect** — фоновая задача демона обнаруживает события USB-портов и устанавливает флаги. Основной цикл ESPHome открывает устройство, считывает дескрипторы, захватывает интерфейсы и кэширует конфигурационные данные.
2. **Disconnect** — интерфейсы освобождаются, дескриптор устройства закрывается, и TCP-соединение USB/IP завершается, чтобы клиент Linux немедленно обнаружил удаление.
3. **Fallback discovery** — если обратный вызов `NEW_DEV` в ESP-IDF не срабатывает (известный особый случай), демон сканирует адреса 1–10 через 5,5 секунды.

### Прошивка HP P1102w

При включении HP LaserJet P1102w работает как USB Mass Storage device ("Smart Install"). Он должен получить блок прошивки перед переключением в режим класса Принтер. Когда `printer_firmware` сконфигурирован, это происходит автоматически:

1. **Detect** — интерфейс Mass Storage (класс 0x08) на HP VID запускает процесс
2. **Mode switch** — команда SCSI vendor 0xD0 отправляется через CBW; устройство отключается и перечисляется заново
3. **Upload** — прошивка отправляется блоками по 64 байта; прогресс регистрируется каждые 10%
4. **Ready** — устройство перечисляется заново с загруженной прошивкой (`FWVER:` появляется в Device ID)

Прошивка энергозависима (сбрасывается при каждом включении), поэтому загрузка повторяется при каждом подключении.

### Отслеживание заданий на печать

Активность bulk OUT отслеживается для предоставления информации о заданиях на печать в логах:

- **Start** — первая bulk OUT передача после 5 секунд простоя
- **Progress** — отправленные байты, количество передач и пропускная способность регистрируются каждые 2 секунды
- **Data preview** — первые 48 байт показаны в ASCII/hex для идентификации формата данных (PJL, ZJS, PCL и т.д.)
- **End** — 5 секунд бездействия после последней передачи

### Синхронизация передач

USB-передачи в ESP-IDF асинхронны. Компонент использует семафорную схему с номерами последовательности для сопоставления завершений с запросами:

1. `prepare_transfer()` присваивает уникальный номер последовательности и очищает состояние завершения
2. Обратный вызов передачи принимает только завершения, соответствующие ожидаемому номеру последовательности
3. `wait_for_transfer()` активно обрабатывает события USB-хоста во время ожидания, предотвращая взаимные блокировки между основным циклом ESPHome и задачей демона USB


## Ограничения

- Только одно USB-устройство одновременно (USB-хост с одним портом)
- Только один клиент USB/IP одновременно
- Изохронные и прерывистые передачи не реализованы (только bulk и control)
- Только устройства Full-Speed (12 Мбит/с) — USB-хост ESP32-S3 не поддерживает High-Speed
- PSRAM должен быть отключен из-за ограничений DMA

## Возможные проблемы и их решение

| Симптом | Причина | Решение                                                                |
|---------|-------|--------------------------------------------------------------------|
| `usbip list` не показывает устройства | Принтер еще не перечислен | подождите 10 секунд после загрузки; проверьте веб-журнал на "Device enumerated"  |
| `usbip attach` зависает | Блокировка брандмауэром порта 3240 | Откройте TCP-порт 3240 в сети                                  |
| Bulk OUT передачи не проходят / таймаут | SG-DMA freeze с большими передачами | Уже решено путем разбиения на MPS-блоки; проверьте журналы на ошибки STALL/NAK |
| `baud_rate` error on boot | Logger using USB pins | Установите `logger: baud_rate: 0`                                         |
| Принтер застрял в режиме mass storage | Прошивка не сконфигурирована или загрузка не удалась | Добавьте `printer_firmware: sihpP1102.dl` в конфигурацию; убедитесь, что файл существует |
| Linux видит устройство, но печать не работает | Неправильный драйвер или формат данных | Используйте `foo2zjs` для HP P1102w; проверьте с помощью `lpstat -t`               |


