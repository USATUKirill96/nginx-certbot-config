[Тут](docs/azure.md) есть инструкция по настройке бесплатной машины от microsoft azure для вашего сервера

[English versuion](README.eng.md) available here

# Скрипт для автоматического деплоя Shadowsocks + v2ray на удаленном сервере и генерации клиенстких конфигов

1. [Что это и для чего](#about)
2. [Что делает скрипт на сервере](#tech-details)
3. [Предварительная настройка](#pre-setup)
4. [Настройка локального окружения](#local-setup)
5. [Конфиги для сервера](#server-setup)
6. [Выгрузка и теги](#deploy)
7. [Настройка клиента](#client)
8. [Типичные ошибки и их исправления](#faq)


# Что это и для чего <a id="about"></a>
- [Shadowsocks](https://shadowsocks.org/en/index.html) - быстрый защищенный и гибкий в настройке прокси-сервер
- [v2ray](https://www.v2ray.com/en/#project-v-) - инструмент для построения коммуникации между клиентской машиной и сервером

Данный скрипт разворачивает Shadowsocks сервер с плагином v2ray, соединение происходит через websocket(https), поэтому
потребуется зарегистрировать домен для вашего сервера.
  
  
Вместе они позволяют построить защищенную и трудную в обнаружении сеть для проксирования запроса и обхода блокировок.
v2ray зарекомендовал себя как отличный инструмент для обхода ограничений в китае, в отличие от VPN соединения он
не так очевиден для провайдера, минимизирует потери в скорости и сокращает расход батареи мобильного телефона.
**v2ray не предназначен для защиты трафика.** В первую очередь, это средство обхода блокировок. Пакеты могут идти 
мимо туннеля, а в случае проблем с сетью ваше устройство скорее всего просто пустит трафик напрямую


# Что сделает скрипт <a id="tech-details"></a>

- На сервере:

  Если кратко, устанавливает докер, открывает порты, запускает в контейнере Nginx, Shadowsocks-rust и контейнер с сертботом. Если более полно, то шаги описаны на [Схеме](docs/server-ru.png)

- На клиенте:

  Генерирует config.json для настройки ss-local клиента и qr-код для настройки мобильных клиентов


# Предварительная настройка <a id="pre-setup"></a>

1. В первую очередь потребуется vps сервер. Достаточно минимальной конфигурации памяти, процессора и накопителя, 
  ориентируйтесь на ежемесячный объем трафика. Google, Amazon и Oracle предоставляют бесплатные VM инстансы сроком от года до пожизненного пользования, список можно посмотреть [тут](https://github.com/ripienaar/free-for-dev#major-cloud-providers).

2. Также для сервера потребуется зарегистрировать домен. Существует множество сервисов, предоставляющих возможность
  бесплатно зарегистрировать домен с автоматическим продлением [список](https://github.com/ripienaar/free-for-dev#dns), [пример](https://www.dnsexit.com/).

# Настройка локального окружения <a id="local-setup"></a>

Вам потребуется:
- Python 3.7
- Virtualenv
  
## Установка python 3.7 

### Для windows:
  1. Скачать [Windows x86-64 executable installer](https://www.python.org/downloads/release/python-370/)
  2. Запустить
  3. Установить

### Для Ubuntu linux
```bash
sudo apt update
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.7 build-essential python3-dev
```

### Для MacOS
```bash
brew install python@3.7
```

## Установка virtualenv
```bash
python3.7 -m pip install virtualenv
```

## Создаем виртуальное окружение и устанавливаем зависимости
Находясь в текущей директории введите
```bash
python3.7 -m virtualenv venv --python 3.7
source venv/bin/activate
pip install -r requirements.txt
```

Впоследствии используйте `source venv/bin/activate' для активации виртуального окружения перед выгрузкой


# Конфиги для сервера <a id="server-setup"></a>

Создайте в корневой директории проекта файл variables.yml (Должен лежать рядом с variables.example.yml) и заполните его по примеру из variables.example.yml:

- user: Имя пользователя на сервере
- host: Домен, оформленный для сервера
- email: Электронная почта
- proxy_password: Пароль для доступа к прокси
- method: Метод шифрования. Можно оставить как в примере
- local_port: Порт, на котором будет работать ваш локальный сервер
- fast_open: true для ускорения работы сервера. Работает на ядрах 3.1+ (на вашем скорее всего работает)
- endpoint: эндпоинт для прокси. Можно поставить что-то наименее подозрительное для провайдера
- enable_firewall: yes для включения фаервола после стартовой настройки. Если на сервере уже есть ваши проекты, и вы не знаете для чего это, включение может вызвать ошибки в их работе

В директории inventories создайте файл `hosts.ini` по образцу из `inventories/hosts.example.ini`

- server ansible_host= Домен, оформленный для сервера или айпи адрес сервера

# Выгрузка и теги <a id="deploy"></a>

## Выгрузка на сервер

Для работы прокси сервера на удаленной машине должен быть установлен docker, docker-compose, а также открыты 80 и 443 порты. Если этого нет, произведите первичную настройку командой

```bash
ansible-playbook -i inventories/hosts.ini --extra-vars "@variables.yml"  deploy/setup.yml 
```

После успешной настройки выгрузите прокси сервер и генератор ключей командой

```bash
ansible-playbook -i inventories/hosts.ini --extra-vars "@variables.yml"  deploy/server.yml   
```

Доступные теги для первичной настройки сервера:
- disable-iptables - Очищает конфиги iptables. Актуально для серверов oracle, которые используют их как фаервол

Доступные теги для выгрузки на сервер:
- nginx - выгрузка конфига nginx. Нужно запустить если решите изменить эндпоинт для прокси-сервера после запуска
- shadowsocks - выгрузка shadowsocks.  Нужно запустить если решите изменить конфиги прокси-сервера
- keys - выгрузка генерации ssl сертификата. Нужно  запустить если сменили адрес хоста для сервера

## Выгрузка на клиенте

```bash
ansible-playbook -i inventories/hosts.ini --extra-vars "@variables.yml"  deploy/client.yml
```

Запустите для генерации конфигов клиентских устройств. В папке `client-config` вы обнаружите следующие файлы:
- shadowsocks-libev.config для настройки клиента ubuntu
- config.json для настройки windows и mac клиентов
- qrcode.png для настройки android и ios клиентов [Источник скрипта](https://github.com/OriginCode/shadowsocks-libev-qrcode)

# Настройка клиента <a id="client"></a>

## Ubuntu linux

Выполните в терминале команды:
```bash
sudo apt update
sudo apt install shadowsocks-libev
sudo apt install shadowsocks-v2ray-plugin
```
Это установит клиентское приложение и плагин из репозитория

Скопируйте клиентский конфиг в папку с конфигом ss-client:
```bash
cp client-config/shadowsocks-libev.json /etc/shadowsocks-libev/config.json
```

Запустите сервер командой
```bash
ss-local
```

## Android

Установите [клиент shadowsocks](https://play.google.com/store/apps/details?id=com.github.shadowsocks) и 
[плагин v2ray](https://play.google.com/store/apps/details?id=com.github.shadowsocks.plugin.v2ray)

В самом клиенте нажмите на иконку с добавлением конфигураций

![](docs/android.png)

и выберите "Сканировать qr код"

Отсканируйте qr код из client-config/qrcode.png

## Windows
- Скачайте последнюю версию [shadowsocks](https://github.com/shadowsocks/shadowsocks-windows/releases) и распакуйте архив
- Скачайте последнюю версию [v2ray](https://github.com/shadowsocks/v2ray-plugin/releases) распакуйте архив, переименуйте файл внутри в v2ray.exe и сохраните в папку с shadowsocks
- Запустите Shadowsocks.exe, заполните поля как в конфиге client-config/shadowsocks-libev.json или импортируйте из него настройки

## iOS
Установите Shadowrocket из магазина приложений и отсканируйте qr код из папки client-config/qrcode.png

# Типичные ошибки и их исправления <a id="faq"></a>

## Сервера Oracle и iptables

Если прокси по какой-то причине не заработал, запустите предварительную настройку и выгрузку заново,
указав тег `disable-iptables` для предварительной настройки. Так же справедливо для серверов других провайдеров

## Firewall от провайдера

Проверьте, что в панели управления  VPS провайдера не установлены дополнительные ограничения
