# TryHackMe - Battery

## 1. Reconnaissance (Разведка)

Первым шагом было сканирование портов с помощью **RustScan** для определения периметра атаки.
```bash
rustscan -a 10.66.135.158 -- -sV
```
**Результат:** Обнаружены открытые порты **22 (SSH)** и **80 (HTTP)**.

![Запуск rustscan](Screens/Battery_Recon_1.png)
![Открытые порты](Screens/Battery_Recon_1.png)

На порту 80 запущен веб-сервер Apache 2.4.7. 

![Веб-страница](Screens/Battery_Web.png)

Был запущен перебор файлов и директорий черезе **FFUF**:

```bash
ffuf -w wordlist/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.66.135.158/FUZZ -e .php,.html,.txt
```

![Перебор файлов и директорий](Screens/Battery_Fuzz.png)

Перейдя на обнаруженный /admin.php - открывается страница входа, а также возможность зарегестрироваться. Но создав пользователя в самой панели мы не найдем ничего интересного, кроме того, что для доступа к /forms.php требуется админ-аккаунт.

![Страница входа](Screens/Battery_Login_Page.png)
![Страница регистрации](Screens/Battery_Register_Page.png)
![Успешная регистрация](Screens/Battery_Register_Successful.png)
![Страница панели](Screens/Battery_Dashboard_Page.png)

## 2. Initial Access (Первоначальный доступ)

Скачал до этого обнаруженный /report обнаружилось, что это исполняемый-скомпилированный файл, но прочитав его обычным **cat** можно увидеть строчки со списком аккаунтов, а также отдельно `admin@bank.a`.

![Чтение report файла](Screens/Battery_Report_File.png)

Попытка зарегистрировать пользователя `admin@bank.a` напрямую выдала ошибку "Nope you are wasting your time :)" (пользователь уже существует).
Однако, в HTML-коде поля uname было ограничение maxlength="12". Это натолкнуло на мысль об уязвимости `SQL Truncation`.

Был отправлен запрос регистрации пользователя `admin@bank.a  '`(добавив пробел в конце и любые символы). База данных при сохранении обрезала лишние символы из-за ограничения размера столбца, и в итоге был перезаписан пароль существующего администратора, что позволило зайти в аккаунт.

![Попытка регистрации админ логина](Screens/Battery_Try_Register_Admin.png)
![Ошибка регистрации](Screens/Battery_Fail_Register_Admin.png)
![Отправка запроса с SQL Truncation](Screens/Battery_Sql_Truncation_Send.png)
![Успешный вход](Screens/Battery_Admin_Success_Login.png)

Перейдя на недоступную до этого /forms.php, открывается страница с возможностью ввода номера аккаунта и сообщения, но при вводе и нажатии отправить - на самой странице ничего не происходит, но если посмотреть в **HTTP History** в **Burp Suite** - можно увидеть POST запрос, который отправляет XML-данные.

![Страница forms.php](Screens/Battery_Form_Page.png)
![Ввод данных и попытка отправить](Screens/Battery_Form_Enter_And_Send.png)
![POST запрос в Burp Suite](Screens/Battery_Burp_Form.png)
![Содержимое запроса](Screens/Battery_Burp_Form_Request.png)

Прежде чем пытаться читать файлы (использовать внешние сущности), было проверено, обрабатывает ли XML-парсер сущности в принципе. Для этого была объявлена **внутренняя сущность** `<!ENTITY test "qwe">` и вызвана в теге `<search>`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY test "qwe"> ]>
<root>
    <name>qweqweqwe</name>
    <search>&test;</search>
</root>
```

![Проверка на обработку сущности](Screens/Battery_Entity_Test.png)

**Результат:** Сервер вернул в ответе `Sorry, account number qwe is not active!`. То, что вместо &test; подставилось слово qwe, подтвердило, что парсер обрабатывает сущности. Значит, уязвимость XXE присутствует, и можно переходить к эксплуатации (чтению файлов).

Поскольку обработка сущностей сработала, был заменен внутренняя сущность на внешнюю (SYSTEM), чтобы прочитать файл /etc/passwd.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
<root>
    <name>qweqweqwe</name>
    <search>&ent;</search>
</root>
```

![Чтение /etc/passwd через XXE](Screens/Battery_XXE_Etc_Passwd.png)

**Результат:** получено содержмое файла /etc/passwd

## 3. Foothold (Закрепление)

Был использован PHP-фильтр с кодированием в base64, чтобы прочитать исходный код найденных до этого PHP файлов, в acc.php были обнаружены учетные данные пользователя `cyber`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY ent SYSTEM "php://filter/convert.base64-encode/resource=acc.php"> ]>
<root>
    <name>qweqweqwe</name>
    <search>&ent;</search>
</root>
```

![Чтение acc.php через XXE](Screens/Battery_acc_read.png)
![Декодирование base64](Screens/Battery_base64_decode_1.png)
![Обнаруженные учетные данные](Screens/Battery_base64_decode_2.png)

Используя найденные креды, удалось подключиться к серверу по SSH.

![Подключение по SSH](Screens/Battery_SSH_Connect.png)

Далее был прочитан user флаг.

![User флаг](Screens/Battery_User_Flag.png)

## 4. Privilege Escalation

В домашней директории пользователя `cyber` находился файл run.py. Он принадлежит пользователю `root` и был недоступен для чтения, но пользователь `cyber` мог запускать его через sudo.

![Проверка на то, какие команды можно выполнять от имени суперпользователя](Screens/Battery_Sudo_l.png)

Был переименован оригинальный скрипт, а затем создан новый run.py с вредоносным содержимым и запущен через sudo, что дало `root`:

![Переименование и создание нового файла](Screens/Battery_Manipulate_With_File.png)
![Получение root](Screens/Battery_Get_Root.png)

Получив root-доступ, были прочитаны финальные флаги: root.txt в /root и flag2.txt в домашней директории пользователя yash.

![Root флаг](Screens/Battery_Root_Flag.png)
![2 флаг](Screens/Battery_Flag_2.png)


## Remediation (Рекомендации по устранению)

Для предотвращения подобных атак в реальной инфраструктуре следует:
1.  **SQL Truncation :** Использовать строгую валидацию длины полей на стороне сервера. В базе данных использовать типы данных с явным ограничением.
2.  **XXE:** Если формат XML не является строгим требованием бизнеса (например, интеграция со старыми SOAP-сервисами), рекомендуется полностью отказаться от него в пользу `JSON`. `JSON` не поддерживает DTD и внешние сущности, что полностью исключает возможность возникновения XXE-уязвимости (Secure by Design).
3.  **Безопасная конфигурация парсера (если XML необходим):** Если использование XML неизбежно, необходимо отключить обработку внешних сущностей (DTD) на уровне парсера.
4.  **Права доступа:** Не предоставлять пользователям права `sudo` на запуск скриптов, которые находятся в директориях с правами на запись для этого пользователя. Использовать абсолютные пути и проверять хэш-суммы скриптов.
5.  **Безопасность данных:** Никогда не хранить пароли в открытом виде в исходном коде.