# Как правильно настроить SSH на Linux

Самая надёжная базовая схема такая: вы создаёте отдельного пользователя, входите по SSH-ключу, а административные действия выполняете через sudo. Root-логин напрямую не используете, а вход по паролю и пароль для использования команд sudo отключаете. Тогда даже если кто-то будет круглосуточно «стучаться» в SSH, он упрётся в отсутствие паролей как класса.

1. Генерация ключа на ПК в Windows PowerShell в папке Downloads

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

Если выдает ошибку, что ключ UNDETECTED 


