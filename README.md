
## Задание
1. Создать две виртуальные машины: **VM1** и **VM2**.
2. Установить на обе (VM1, WM2) Astra Linux
3. На **VM1** поднять службы:
   - SSH-сервер
   - Samba + libpam-mount
   - PostgreSQL
4. На **VM2** поднять службы:
   - SSH-сервер
   - fly-dm
5. Между VM1 и VM2 поднять сеть 192.168.20.0/24
6. На **VM1** настроить PostgreSQL:
   - Перенаправить логи в `/var/log/postgresql`.
   - Включить cert.
   - Добавить cert в pg_hba.conf.
7. На **VM1** создать пустую базу данных и разрешить подключение к ней только по сертификату и только для сети `192.168.20.0/24`.
8. Авторизоваться под пользователем admin на VM2 и подключиться к БД расположенной на VM1 (подключение к БД делать только через консоль под пользователем admin с сертификатом).
9. На WM2 при входе в ОС под пользователем admin должно происходить автоматическое монтирование удаленной директории с VM1 (используем pam_mount).

---


### Создание виртуальных машин
- Для создания ВМ я использовал VMware Workstation
- Созданны следующие виртуальные машины:
- **Astra1**:
  - ОЗУ: 2 ГБ
  - CPU: 2 ядра
  - HDD: 20 ГБ
  - Сетевые интерфейсы:
    - **Bridge**: Для подключения к локальной сети.
    - **Host-only**: В подсети `192.168.20.0/24`

- **Astra2**:
  - ОЗУ: 2 ГБ
  - CPU: 2 ядра
  - HDD: 20 ГБ
  - Сетевые интерфейсы:
    - **Bridge**: Для подключения к локальной сети.
    - **Host-only**: В подсети `192.168.20.0/24`

---

### Установка Astra Linux
- На обе машины был установлен Astra Linux Special Edition 1.8 с уровнем защищённости «Орёл» Поскольку использовалась версия 1.8, вместо пользователя admin по умолчанию был создан пользователь administrator, который будет использоваться для всех дальнейших операций.

---

### Настройка сети:

```bash
sudo nano /etc/network/interfaces
```
Astra1
```ini
auto enp0s3
iface enp0s3 inet static
    address 192.168.20.2
    netmask 255.255.255.0
```

Astra2
```ini
auto enp0s3
iface enp0s3 inet static
    address 192.168.20.3
    netmask 255.255.255.0
```

Переапуск сети:
```bash
sudo systemctl restart networking
```

### Настройка Astra1

#### Установка пакетов:
   ```bash
   sudo apt update
   sudo apt install openssh-server samba libpam-mount postgresql -y
   ```

#### Создание общей папки:
   ```bash
   sudo mkdir -p /srv/samba/share
   sudo chmod 777 /srv/samba/share
   ```
#### Настройка Samba:
   ```bash
   sudo nano /etc/samba/smb.conf
   ```

   ```ini
   [share]
      path = /srv/samba/share
      browseable = yes
      writable = yes
      create mask = 0777
      directory mask = 0777
   ```

   ```bash
   sudo systemctl restart smbd
   ```

#### Создаем директорию для логов:
   ```bash
   sudo mkdir -p /var/log/postgresql
   sudo chown postgres:postgres /var/log/postgresql
   ```
   Изменяем файл конфигурации:
   ```bash
   sudo nano /etc/postgresql/15/main/postgresql.conf
   ```

   ```ini
   log_directory = '/var/log/postgresql'
   ```
#### Создаем сертификаты:
   ```bash
   sudo openssl req -new -x509 -days 365 -nodes -text -out /etc/ssl/certs/server.crt -keyout /etc/ssl/private/server.key -subj "/CN=vm1"
   sudo chmod 600 /etc/ssl/private/server.key
   sudo chown postgres:postgres /etc/ssl/private/server.key /etc/ssl/certs/server.crt
   ```
#### Настройка PostgreSQL для использования сертификатов:
   ```bash
   sudo nano /etc/postgresql/15/main/postgresql.conf
   ```

   ```ini
   ssl = on
   ssl_cert_file = '/etc/ssl/certs/server.crt'
   ssl_key_file = '/etc/ssl/private/server.key'
   ```
#### Настройка `pg_hba.conf`:
   ```bash
   sudo nano /etc/postgresql/15/main/pg_hba.conf
   ```

   ```ini
   hostssl all all 192.168.20.0/24 cert
   ```
#### Создаем базу данных и пользователя:
   ```bash
   sudo -u postgres psql -c "CREATE DATABASE testdb;"
   sudo -u postgres psql -c "CREATE USER administrator WITH PASSWORD 'adminpassword';"
   sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE testdb TO administrator;"
   ```

   ```bash
   sudo systemctl restart postgresql
   ```

---

### Настройка Astra2

#### Установка пакетов:
   ```bash
   sudo apt update
   sudo apt install openssh-server libpam-mount fly-dm cifs-utils -y
   ```


#### Настройка `pam_mount`:
   ```bash
   sudo nano /etc/security/pam_mount.conf.xml
   ```

   ```xml
   <volume user="administrator" fstype="cifs" server="192.168.20.1" path="share" mountpoint="~/mnt/share" options="username=administrator,password=adminpassword,rw" />
   ```
Перезапускаем службу:
   ```bash
   sudo systemctl restart fly-dm
   ```

#### Подключение к PostgreSQL с VM2
1. Создаем сертификаты:
   ```bash
   openssl req -new -x509 -days 365 -nodes -text -out admin.crt -keyout admin.key -subj "/CN=administrator"
   chmod 600 admin.key
   ```
2. Пробрасоваем cert:
   ```bash
   scp admin.crt admin.key administrator@192.168.20.2:/home/administrator/
   ```
3. Подключаемся к базе данных:
   ```bash
   psql "host=192.168.20.2 dbname=testdb user=administrator sslmode=require sslcert=admin.crt sslkey=admin.key"
   ```

---

#### Настройка автоматического монтирования:
Редактируем `pam_mount`:
   ```bash
   sudo nano /etc/security/pam_mount.conf.xml
   ```

   ```xml
   <volume user="administrator" fstype="cifs" server="192.168.20.1" path="share" mountpoint="~/mnt/share" options="username=administrator,password=adminpassword,rw" />
   ```

   ```bash
   sudo systemctl restart fly-dm
   ```

---

