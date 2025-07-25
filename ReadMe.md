# Дипломная работа

## Track Penetration Testing (Исследование защищённости информационной системы методом чёрного ящика)

---

## Информация, вводные данные:

Предоставлен IP тестируемого приложения (сервиса) - **92.51.39.106**

Необходимо протестировать приложение на безопасность – провести полноценное тестирование на проникновение методом черного ящика.

Тестирование необходимо провести в рамках пентеста приложения, с использованием open source инструментов, рассмотренных в курсе.

## Этап 1. OSINT

Для поиска информации о предоставленном IP адресе были использованы:

* [Shodan.io](https://www.shodan.io/) - поисковая система, позволяющая пользователям искать различные типы серверов (веб-камеры, маршрутизаторы, серверы и так далее), подключённых к сети Интернет, с использованием различных фильтров.
* [Criminalip.io](https://www.criminalip.io/) - это поисковая система интеллекта в области киберугроз (CTI), предназначенная для выявления потенциальных уязвимостей, которые угрожают ИТ-активам компаний или отдельных лиц и предлагают новый способ управления ими всесторонне, позволяя пользователям находить результаты для вредоносного IP-адреса, доменов, уязвимостей, баннеров и другой информации, связанной с безопасностью.
* [Google.com](https://google.com) - поисковая система интернета. Нас интересует использование Google Dorking.

### 1.1. Результат сбора информации с помощью Shodan.io

Веб-отчет Shodan.io находится здесь - [https://www.shodan.io/host/92.51.39.106](https://www.shodan.io/host/92.51.39.106)

В результате сканирования получена следующая информация:

| # | Параметры         | Результат               |
| - |-------------------|-------------------------|
| 1 | Hostnames         | 1427771-cg36175.tw1.ru  |
| 2 | Domains           | tw1.ru                  |
| 3 | Country           | Russian Federation      |
| 4 | City              | Saint Petersburg        |
| 5 | Organization      | TimeWeb Ltd.            |
| 6 | ISP               | TimeWeb Ltd.            |
| 7 | ASN               | AS9123                  |
| 8 | Operating System  | Linux                   |

Найдены открытые порты:

* 22 (TCP) - OpenSSH8.2p1 Ubuntu 4ubuntu0.13
* 7788 (TCP) - Server: TornadoServer/5.1.1
* 8050 (TCP) - Server: Apache/2.4.7 (Ubuntu)

Скриншоты интересующих нас данных:

![](assets/1-osint/1-shodan-general.jpg)

![](assets/1-osint/1-shodan-web-technology.jpg)

### 1.2. Результат сбора информации с помощью Criminalip.io

Веб отчет Criminalip.io находится здесь - [https://www.criminalip.io/asset/report/92.51.39.106](https://www.criminalip.io/asset/report/92.51.39.106)

В результате сканирования получена следующая информация:

| # | Параметры         | Результат          |
| - |-------------------|--------------------|
| 1 | ASN               | 9123               |
| 2 | AS Name           | TimeWeb Ltd.       |
| 3 | Organization Name | TimeWeb            |
| 4 | Country Code      | RU                 |
| 5 | Country           | Russian Federation |
| 6 | Region            | St.-Petersburg     |
| 7 | City              | St Petersburg      |
| 8 | Postal Code       | 195213             |

Найдены открытые порты:

* 22 (TCP) - SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.13

Более открытых портов Criminalip.io не показал, что странно, так выглядит текущий отчет:

![](assets/1-osint/1-criminalip-all-info.jpg)

### 1.3. Результат сбора информации с помощью Google Dorking

* ввод в поле поиска `site:92.51.39.106` выдает информацию:

    ![](assets/1-osint/1-google-dork_1.jpg)

* ввод в поле поиска `inurl:admin site:92.51.39.106` выдает:

    ![](assets/1-osint/1-google-dork_2.jpg)

* ввод в поле поиска `filetype:log OR filetype:txt site:92.51.39.106` не выдает результатов:

  ![](assets/1-osint/1-google-dork_3.jpg)

По результату Google Dorking:

* Скрытые страницы не найдены.
* Конфиденциальные файлы не обнаружены.

### 1.4. Анализ собранной информации

На основе полученной информации, определяем цели для атаки:

* [http://92.51.39.106:8050/](http://92.51.39.106:8050/) - NetologyVulnApp.com, в качестве веб-сервера используется Apache/2.4.7
* [http://92.51.39.106:7788/](http://92.51.39.106:7788/) - Beemers, в качестве веб-сервера используется TornadoServer/5.1.1

Также выявлено использование устаревшей версии PHP 5.5.9 на сервере. На сайте `cvedetails.com` [найдено большое количество](https://www.cvedetails.com/version/1334460/PHP-PHP-5.5.9.html)
уязвимостей данной версии:

![](assets/1-osint/1-php559-vulnerabilities.jpg)

Версия TornadoServer 5.1.1 также является устаревшей и имеет ряд уязвимостей, о которых есть информация на сайте `cvedetails.com`:

![](assets/1-osint/1-tornado-web.jpg)

И так, когда у нас выявлены уязвимые сервисы, работающие на определенных портах, можно переходить к следующему этапу - сканированию.

## Этап 2. Scanning

### 2.1. Сканирование NMAP

Сканирование начну с утилиты nmap.

У меня имеется заранее подготовленная виртуальная машина Kali Linux, выполняю команду в терминале `nmap -sV -O -A 92.51.39.106`:

![](assets/2-scan/2-nmap.jpg)

Результат:

* Открыт порт 22 - OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
* Операционная система - Linux 5.0 - 5.14

### 2.2. Сканирование ZAP Docker

В виртуальной машине Kali установил необходимый образ в соответствии с [инструкцией](https://www.zaproxy.org/docs/docker/about/) с официального сайта
командой `docker pull ghcr.io/zaproxy/zaproxy:stable`:

![](assets/2-scan/2-zap-docker-pulled.jpg)

Поочередно буду сканировать адрес 92.51.39.106 по портам 8050 и 7788.

#### Сканирование адреса `http://92.51.39.106:8050`:

Запускаю сканирование по порту 8050 командой `docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
    -t http://92.51.39.106:8050 -g gen.conf -r Report_8050.html`

#### Сканирование адреса `http://92.51.39.106:7788`:

Запускаю сканирование по порту 7788 командой `docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
    -t http://92.51.39.106:7788 -g gen.conf -r Report_7788.html`

По результату сканирования получен [отчет](assets/2-scan/zap-docker/Report_7788.html).

Суммарный отчет по результату сканирования:

![](assets/2-scan/2-zap-scan-7788-completed.jpg)

Перечислим полученные High Priority Alerts:

* Cross Site Scripting (DOM Based)
* Cross Site Scripting (Reflected)
* Path Traversal
* Remote OS Command Injection
* SQL Injection

### 2.3. Сканирование ZAP с графической оболочкой

Ранее я уже установил ZAP на виртуальную машину Kali Linux.

#### 2.3.3 Сканирование адреса `http://92.51.39.106:7788`:

![](assets/2-scan/2-zap-scan-7788.jpg)

По результату сканирования получен [отчет](assets/2-scan/zap/2025-07-25-ZAP-Report-7788.html).

Перечислим полученные High Priority Alerts:

* Cross Site Scripting (DOM Based)
* Cross Site Scripting (Reflected)
* Path Traversal
* Remote OS Command Injection
* SQL Injection

Скриншот окна ZAP после завершения сканирования `92.51.39.106:7788`:

![](assets/2-scan/2-zap-scan-7788-completed.jpg)

#### 2.3.4 Сканирование адреса `http://92.51.39.106:8050`:
