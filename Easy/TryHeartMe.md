# TryHackMe - TryHeartMe

## 1. Reconnaissance (Разведка)

Первым шагом было сканирование портов с помощью **RustScan** для определения периметра атаки.

```bash
rustscan -a 10.64.163.130 -- -sV
```
**Результат:** Обнаружены открытые порты 22 (SSH) и 5000 (HTTP).

![Запуск rustscan](Screens/TryHeartMe_Recon_1.png)
![Открытые порты](Screens/TryHeartMe_Recon_2.png)

На порту 5000 запущен веб-сервер Werkzeug 3.0.1.

![Страница веб-приложения](Screens/TryHeartMe_Main_Page.png)

Был запущен перебор файлов и директорий через **FFUF**:

```bash
ffuf -w wordlist/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.64.163.130:5000/FUZZ -e .php,.html,.txt
```

![Перебор файлов и директорий](Screens/TryHeartMe_Fuzz.png)

**Результат:** Обнаружена скрытая директория администратора - /admin.

Перейдя на /admin - происходит редирект на страницу логина.

![Редирект на логин](Screens/TryHeartMe_Admin_Redirect.png)

## 2. Exploitation (Эксплуатация)

На странице была возможность зарегистрироваться. Создав аккаунт и попытавшись перейти на /admin, была получена ошибка 403 Forbidden.

![Создание аккаунта](Screens/TryHeartMe_Create_Account.png)
![Ошибка 403](Screens/TryHeartMe_Admin_403.png)

С помощью расширения **JWT Editor** в **Burp Suite** было замечено (подсвечено), что сервер использует JWT-токен для аутентификации. Для удобства экспериментов использована страница /account, где изменения в токене сразу отображаются в ответе сервера.

![Создание аккаунта](Screens/TryHeartMe_Account_Page.png)
![Запрос в Burp Suite](Screens/TryHeartMe_Account_Page_Burp.png)

Использовав атаку **None Algorithm (подмена алгоритма подписи на none)**, было замечено, что сервер принимает такой токен без проверки подписи и возвращает тот же ответ 200 OK, что и для валидного JWT.

![Использование атаки "none" алгоритма](Screens/TryHeartMe_Account_Page_Burp_JWT_None.png)
![Ответ на атаку](Screens/TryHeartMe_Account_Page_Burp_JWT_None_Response.png)

Осознав, что сервер доверяет данным внутри токена без проверки подписи, был изменен `payload`: установлена `role: "admin"`. После отправки модифицированного токена в запросе к /admin, я получил успешный доступ к панели администратора.

![Запрос в /admin с обычным JWT](Screens/TryHeartMe_Admin_Page_Burp.png)
![Использование атаки "none" алгоритма](Screens/TryHeartMe_Admin_Page_Burp_JWT_None.png)
![Ответ на атаку](Screens/TryHeartMe_Admin_Page_Burp_JWT_None_Response.png)

Используя подделанный cookie в браузере, был обнаружен эксклюзивный товар "ValenFlag". Совершив покупку, был получен флаг.

![Страница админ](Screens/TryHeartMe_Admin_ValenFlag.png)
![Покупка ValenFlag](Screens/TryHeartMe_Admin_ValenFlag_Buy.png)
![Флаг](Screens/TryHeartMe_Flag.png)

## Remediation (Рекомендации по устранению)

Для предотвращения подобных атак необходимо:

1. **Безопасность JWT:** Сервер должен принудительно проверять алгоритм подписи (например, только `HS256`). Токены с алгоритмом `none` или невалидной подписью должны немедленно отклоняться.
2. **Контроль доступа:** Никогда не полагаться на поля из JWT (например, `role: "admin"`) для принятия решений об авторизации. Сервер должен проверять права пользователя в базе данных при каждом запросе.