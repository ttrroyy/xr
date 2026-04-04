# Multi гайд по настройке для VLESS REALITY на Ubuntu:

[Как правильно настроить SSH на Linux](#Как-правильно-настроить-SSH-на-Linux)
## Как правильно настроить SSH на Linux

Самая надёжная базовая схема такая: создаем отдельного пользователя, входим по SSH-ключу, а административные действия выполняем через sudo. Root-логин напрямую не используем, а вход по паролю и пароль для использования команд sudo отключаем. Тогда даже если кто-то будет круглосуточно «стучаться» в SSH, он упрётся в отсутствие паролей как класса.

1. Генерация публичного (.pub) и приватного ключей на ПК в Windows PowerShell в папке Downloads

```
ssh-keygen -t rsa -b 4096 -f "$HOME\Downloads\id_rsa"
```
2. Настройка нового пользователя (замените USER_NAME на имя вашего пользователя) на вашем сервере

```
sudo adduser USER_NAME
sudo usermod -aG sudo USER_NAME
sudo bash -c 'echo "USER_NAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/USER_NAME'
sudo chmod 440 /etc/sudoers.d/USER_NAME
```
3. Подготовка папки ключей и выдача прав (замените USER_NAME на имя вашего пользователя)

```
sudo mkdir -p /home/USER_NAME/.ssh
sudo touch /home/USER_NAME/.ssh/authorized_keys
sudo chown -R USER_NAME:USER_NAME /home/USER_NAME/.ssh
sudo chmod 700 /home/USER_NAME/.ssh
sudo chmod 600 /home/USER_NAME/.ssh/authorized_keys
```

4. Добавление/Замена ключа. Вставьте ваш публичный ключ (содержимое файла .pub с ПК) в редактор:

```
sudo nano /home/USER_NAME/.ssh/authorized_keys
```

5. Перезагружаем SSH 

```
sudo systemctl restart ssh
```
6. Через Windows PowerShell проверяем доступ к серверу по добавленному ключу (обязательно! чтобы не закрыть себе доступ)

```
ssh -i "KEY_PATH" USER_NAME@SERVER_IP
```

Если выдает ошибку, что ключ UNDETECTED нужно отключить наследование для приватного (!!! НЕ .pub) ключа:

![описание](./assets/1.jpg)

![описание](./assets/2.png)

![описание](./assets/3.jpg)

![описание](./assets/4.png)

![описание](./assets/5.png)

![описание](./assets/6.jpg)

*Вместо WINDOWS_USER_NAME пишем имя вашего юзера Windows*

![описание](./assets/7.jpg)

![описание](./assets/8.jpg)

7. Если вы смогли зайти по SSH отключаем пароли и ROOT-вход

```
sudo sed -i -E 's/^#?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo sed -i -E 's/^#?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo sed -i -E 's/^#?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo sed -i -E 's/^#?KbdInteractiveAuthentication.*/KbdInteractiveAuthentication no/' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/50-cloud-init.conf
```

!!! в папке **/etc/ssh/sshd_config.d/** могут быть и другие файлы .conf, созданные различными программами, поэтому если команда сверху не сработала то имйте ввиду, что какой-то конфиг берет приоритет над **50-cloud-init.conf**

Перезагружаем SSH и проверяем итоговые значения:
```
sudo systemctl restart ssh
```
```
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication'
```

Вывод должен быть:
```
permitrootlogin no
passwordauthentication no 
pubkeyauthentication yes
kbdinteractiveauthentication no
```

# Настройка Firewall

1. Ставим и включаем

```
sudo apt update
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

2. Рекомендуется открывать только нужные порты. Также можно открыть порт только для вашего айпи (например для безопасного доступа к панели)

Открытие обязательных портов. Другие добавляются по такому же примеру:
```
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Открытие порта для определенного айпи:

```
sudo ufw allow from ВАШ_IP to any port ВАШ_ПОРТ proto tcp
```

Включение UFW:

```
sudo ufw enable
```

Проверка:
```
sudo ufw status
```

Закрытие порта:
```
sudo ufw delete allow ВАШ_ПОРТ/tcp
```
```
sudo ufw delete allow from ВАШ_IP to any port ВАШ_ПОРТ proto tcp
```

Выключение UFW:
```
sudo ufw disable
```

# Fail2Ban. Защита от перебора и «шума» в логах

Даже при входе по ключам полезно включить защиту от перебора. Он автоматически блокирует IP, которые пытаются ломиться в SSH/RDP/веб-авторизацию. В итоге меньше мусора в логах, меньше попыток подобрать что-либо и проще заметить настоящую проблему.

1. Установка:
```
sudo apt update
sudo apt install -y fail2ban
```

2. Создать локальный конфиг:
```
sudo nano /etc/fail2ban/jail.local
```

3. Пример конфига:
```
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime  = 1h
```

4. Запуск:
```
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

5. Остановка сервиса и отключение автозапуска:
```
sudo systemctl stop fail2ban
```
```
sudo systemctl disable fail2ban
```

6. Разбан IP
```
sudo fail2ban-client set sshd unbanip IP
```

# Настройка DNS в системе

В XRAY мы в любом случае не будем использовать системный днс, но на некоторых хостингах вписаны только локальные днс, что может повысить пинг в системе, а также ограничить доступ к некоторым репозиториям(особенно на серверах с ТСПУ). Поэтому мы пропишем нормальные днс в Global.

```
sudo nano /etc/systemd/resolved.conf
```

В строке #DNS= удаляем # и вписываем 1.1.1.1 8.8.8.8
Должно получиться: DNS=1.1.1.1 8.8.8.8

Нажимаем Ctrl+O, Enter, Ctrl+X

Перезагружаем:
```
sudo systemctl restart systemd-resolved
```

Проверяем:
```
resolvectl status
```
