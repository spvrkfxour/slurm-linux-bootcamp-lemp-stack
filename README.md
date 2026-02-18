## Задание
Развернуть CMS WordPress, но сделать это в защищенной конфигурации - стек LEMP (Linux, Nginx, MariaDB, PHP).

Архитектура решения:
* Данные: Лежат на отказоустойчивом RAID-массиве (LVM тома site_files и db_data)
* Безопасность: Доступ к админ-панелям (phpMyAdmin) разрешен только с вашего IP
* Шифрование: Весь трафик идет через HTTPS
* Бэкап: Выполняется от имени специального пользователя БД с ограниченными правами

## Решение
Поднимем Vagrant и подключемся по SSH
```console
vagrant up
vagrant ssh
```
Создадим RAID 1-массив с LVM томами разделив приложение по томам
```console
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
sudo pvcreate /dev/md0
sudo vgcreate data_vg /dev/md0
sudo lvcreate -L 4G -n lv_web data_vg
sudo lvcreate -L 4G -n lv_db data_vg
sudo lvcreate -l 100%FREE -n lv_log data_vg
```
Отформатируем все тома в ext4 и смонтируем. Для монтирования папки с логами (/var/log) нужно сделать миграцию, так как она уже существует
```console
sudo mkfs.ext4 /dev/data_vg/lv_web
sudo mkfs.ext4 /dev/data_vg/lv_db
sudo mkfs.ext4 /dev/data_vg/lv_log
sudo mkdir /mnt/new_logs
sudo mkdir -p /var/www
sudo mkdir -p /var/lib/mysql
sudo mount /dev/data_vg/lv_web /var/www
sudo mount /dev/data_vg/lv_db /var/lib/mysql
sudo mount /dev/data_vg/lv_log /mnt/new_logs/
sudo systemctl stop rsyslog
sudo cp -a /var/log/. /mnt/new_logs/
sudo umount /mnt/new_logs/
sudo mount /dev/data_vg/lv_log /var/log
sudo systemctl start rsyslog
sudo systemctl daemon-reload
sudo nano /etc/fstab
/dev/data_vg/lv_web /var/www ext4 defaults 0 2
/dev/data_vg/lv_db /var/lib/mysql ext4 defaults 0 2
/dev/data_vg/lv_log /var/log ext4 defaults 0 2
sudo reboot
```
Установим нужные пакеты и настроим стандартно
```console
sudo apt update && sudo apt upgrade
sudo apt install nginx mariadb-server php-fpm php-mysql -y
sudo systemctl enable --now nginx mariadb
sudo nano /etc/nginx/sites-available/default
location ~ \.php$ {
include snippets/fastcgi-php.conf;
fastcgi_pass unix:/run/php/php8.3-fpm.sock;
sudo nginx -t
sudo systemctl reload nginx
sudo mysql_secure_installation
```
Создадим:
```console
sudo mariadb
```
Базу данных
```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
Пользователя для приложения (полные права на базу wordpress)
```sql
CREATE USER 'wp_app'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_app'@'localhost';
```
Пользователя для бэкапов (права только на чтение и блокировку для дампа)
```sql
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'BackupPass456!';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backup_user'@'localhost';
```
Применяем права и выходим
```sql
FLUSH PRIVILEGES;
EXIT;
```
Установим WordPress
```console
# Идем в корень веб-сервера (наш LVM том)
cd /var/www/html
# Удаляем заглушку Nginx
sudo rm index.nginx-debian.html
# Скачиваем и распаковываем
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
# Переносим файлы из подпапки в корень
sudo mv wordpress/* .
sudo rmdir wordpress
sudo rm latest.tar.gz
```
И phpMyAdmin
```console
# Скачиваем
sudo wget https://files.phpmyadmin.net/phpMyAdmin/5.2.3/phpMyAdmin-5.2.3-all-languages.tar.gz
sudo tar -xzvf phpMyAdmin-5.2.3-all-languages.tar.gz
sudo rm phpMyAdmin-5.2.3-all-languages.tar.gz
# Переименовываем для удобства в pma
sudo mv phpMyAdmin-5.2.3-all-languages pma
# Создаем конфиг из шаблона
sudo cp pma/config.sample.inc.php pma/config.inc.php
# Генерируем blowfish_secret и вставляем в конфиг
cd pma/
openssl rand -base64 32
sudo nano config.inc.php
```
Проверяем сервисы и настраиваем права для папок и файлов нашего сервиса
```console
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
sudo find /var/www/html/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/ -type f -exec chmod 644 {} \;
```
Настроим Nginx<br>
Сгенерируем самоподписной сертификат
```console
sudo mkdir -p /etc/nginx/ssl
# В Common Name - mysite.local
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
И настроим конфиг
```console
sudo nano /etc/nginx/sites-available/wordpress
```
```nginx
server {
 listen 80;
 server_name mysite.local;
 return 301 https://$host$request_uri;
}
server {
 listen 443 ssl;
 server_name mysite.local;
 ssl_certificate /etc/nginx/ssl/nginx.crt;
 ssl_certificate_key /etc/nginx/ssl/nginx.key;
 root /var/www/html;
 index index.php index.html;
 # Главный локейшн для WordPress
 location / {
 try_files $uri $uri/ /index.php$is_args$args;
 }
 # Обработка PHP
 location ~ \.php$ {
 include snippets/fastcgi-php.conf;
 fastcgi_pass unix:/run/php/php8.3-fpm.sock;
 }
 # Безопасность phpMyAdmin (ACL)
 location ^~ /pma/ {
 alias /var/www/html/pma/;
 index index.php;
 # Разрешаем только свой ip
 allow 192.168.56.1;
 deny all;
 location ~ \.php$ {
 include snippets/fastcgi-php.conf;
 fastcgi_pass unix:/run/php/php8.3-fpm.sock;
 fastcgi_param SCRIPT_FILENAME $request_filename;
 }
 }
 # Запрет доступа к скрытым файлам (.htaccess, .git)
 location ~ /\. {
 deny all;
 }
}
```
Активируем
```console
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
И создадим скрипт для бэкапа
```bash
sudo nano /usr/local/bin/backup_mysite.sh
sudo chmod +x /usr/local/bin/backup_mysite.sh
#!/bin/bash

BACKUP_DIR="/var/backups/mysite"
SOURCE_DIR="/var/www/html"
RETENTION_DAYS=7
DATE=$(date +%F_%H-%M)

MYSQL_USER="backup_user"
MYSQL_PASS="BackupPass456!"
MYSQL_HOST="localhost"

LOG_FILE="/var/log/backup.log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

mkdir -p $BACKUP_DIR

log "=== Начало бэкапа: $DATE ==="
log "Бэкап файлов из $SOURCE_DIR..."
tar -czf $BACKUP_DIR/files_$DATE.tar.gz $SOURCE_DIR 2>/dev/null
if [ $? -eq 0 ]; then
  log "Файлы успешно забэкаплены: files_$DATE.tar.gz"
else
  log "Ошибка при бэкапе файлов"
fi

log "Бэкап баз данных пользователем $MYSQL_USER..."
mysqldump --user="$MYSQL_USER" --password="$MYSQL_PASS" --host="$MYSQL_HOST" \
--all-databases --single-transaction --quick \
--lock-tables=false | gzip > $BACKUP_DIR/db_$DATE.sql.gz 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
  log "База wordpress успешно забэкаплена: wordpress_$DATE.sql.gz"
else
  log "Ошибка при бэкапе базы wordpress"
fi

log "Очистка бэкапов старше $RETENTION_DAYS дней..."
find $BACKUP_DIR -type f -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -type f -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

log "Размер директории бэкапов:"
du -sh $BACKUP_DIR | tee -a "$LOG_FILE"

log "=== Бэкап завершен ==="
```
```console
sudo crontab -e
*/5 * * * * /usr/local/bin/backup_mysite.sh
sudo systemctl status cron
```
## Результат
<img width="1917" height="981" alt="image" src="https://github.com/user-attachments/assets/d9ea551f-b021-4531-b1ad-06f6c4dac164" />
