# 🛡️ Vaultwarden, Caddy, Fail2ban, Vaultwarden Backup

Инструкция по развёртыванию своего менеджера паролей на облачном сервере для работы с клиентами Bitwarden.

- Vaultwarden — легковесный сервер на Rust, совместимый с приложениями Bitwarden, на вашем облачном сервере данные хранятся в зашифрованном виде
- Caddy — обратный прокси с автоматическим HTTPS
- Fail2ban — защита от перебора паролей
- Vaultwarden Backup — регулярные бэкапы в облачное хранилище

## 🚀 Установка

1. Firewall

   ```bash
   sudo ufw --force enable && sudo ufw allow OpenSSH && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp
   ```

2. Сгенерировать ADMIN_TOKEN

   ```bash
   docker run --rm -it vaultwarden/server /vaultwarden hash
   ```

3. Создать и заполнить .env

   ```dotenv
   DOMAIN=https://subdomain.domain.com
   ACME_EMAIL=admin@example.com
   ADMIN_TOKEN="сгенерированный токен из 2 пункта"
   ```

4. Создать пароль приложения (облачного хранилища) с типом WebDAV

5. Настроить remote для бэкапа

   ```bash
   docker run --rm -it -v ./rclone-data:/config/ ttionya/vaultwarden-backup rclone config
   ```
   - name: yandex-disk
   - type: webdav
   - url: https://webdav.yandex.ru
   - vendor: other
   - user: login@yandex.ru
   - pass: пароль приложения

   > ⚠️ **Восстановление бэкапа**
   >
   > Сделайте это сейчас — см. раздел [«Восстановление из бэкапа»](#-восстановление-из-бэкапа) — и только потом переходите к следующему шагу

6. Проверить подключение

   ```bash
   docker run --rm -v ./rclone-data:/config/ ttionya/vaultwarden-backup rclone lsd yandex-disk:
   ```

7. Запустить всё

   ```bash
   docker compose up -d
   ```

8. Открыть `https://subdomain.domain.com`, создать аккаунт

   > ⚠️ **Если бэкап уже восстановлен**
   >
   > Этот шаг не нужен — аккаунт восстановится вместе с базой

9. В `docker-compose.yml` поставить `SIGNUPS_ALLOWED: "false"`

10. Внести необходимые настройки в панели администратора

11. Для закрытия настроек администратора в `.env` очистить переменную `ADMIN_TOKEN`

12. Перезапустить `docker compose up -d`

13. Проверить fail2ban

    ```bash
    docker compose exec fail2ban fail2ban-client status vaultwarden
    ```

14. Прогнать тестовый бэкап
    ```bash
    docker compose run --rm vaultwarden-backup backup
    ```

## 💾 Восстановление из бэкапа

1. Запустить восстановление, указав те же rclone-параметры, что и в `docker-compose.yml`

   ```bash
   docker run --rm -it \
     -v ./vw-data:/bitwarden/data/ \
     -v ./rclone-data:/config/ \
     -e RCLONE_REMOTE_NAME=yandex-disk \
     -e RCLONE_REMOTE_DIR=/VaultwardenBackup \
     ttionya/vaultwarden-backup restore
   ```

   Контейнер покажет список доступных бэкапов в облачном хранилище.

2. Проверить, что файлы в `./vw-data` восстановились (появились `db.sqlite3`, `rsa_key*` и т.д.)
