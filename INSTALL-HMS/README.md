## Hive Metastore (standalone 4.2.0) для Trino + HDFS — инструкция (Markdown)

Инструкция по установке и настройке standalone Hive Metastore (версия 4.2.0) с PostgreSQL как бэкендом, и интеграции его с Trino, который читает/пишет данные в HDFS. HDFS уже развернут, Trino будет на тех же ВМ, сеть между узлами доступна.

---

## 1. Требования
- Linux (Ubuntu/Debian/CentOS).  
- Java 21+ на машине metastore (для apache-hive-metastore-4.2.0 необходимв JAVA 21)  
- PostgreSQL (локально или удалённо).  
- Hadoop клиентские конфиги: core-site.xml и hdfs-site.xml.  
- JDBC-драйвер PostgreSQL.  
- Порт 9083 открыт для Trino.

---

## 2. Установка PostgreSQL (пример для Ubuntu)
```bash
sudo apt update &&
sudo apt install -y postgresql postgresql-contrib
sudo -u postgres psql <<SQL
CREATE DATABASE hive_metastore;
CREATE USER hive WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE hive_metastore TO hive;
GRANT CONNECT ON DATABASE hive_metastore TO hive;
\c hive_metastore
ALTER SCHEMA public OWNER TO hive;
SQL
```

---

## 3. Скачивание hive-standalone-metastore-4.2.0 и его настройка
### 3.1 Загрузка apache-hive-metastore-4.2.0
(вариант: официальный дистрибутив или артефакт JAR)
```bash
# пример: распакованный дистрибутив
wget https://archive.apache.org/dist/hive/hive-standalone-metastore-4.2.0/hive-standalone-metastore-4.2.0-bin.tar.gz &&
sudo tar -xzf hive-standalone-metastore-4.2.0-bin.tar.gz -C /opt &&
sudo ln -s /opt/apache-hive-metastore-4.2.0-bin /opt/hive &&
sudo mkdir -p /var/log/hive &&
sudo chown -R $(whoami):$(whoami) /opt/apache-hive-metastore-4.2.0-bin /var/log/hive /opt/hive
```
или если у вас готовый standalone JAR:
```bash
# поместите hive-standalone-metastore-4.2.0.jar в /opt/hive
```
### 3.2 Установка и настройка JAVA 21 для работы apache-hive-metastore-4.2.0

Можно иметь на одной машине одновременно несколько JVM, но мы используем для HDFS (Java 21) и для Hive Metastore (Java 21). Если еще не установлена, то откройте страницу с загрузкой дистрибутивов [JDK 21][java], скопируйте ссылку на дистрибутив, загрузите и распакуйте

[java]: https://jdk.java.net/java-se-ri/21

```bash
## загружаем дистрибутив для установки

cd /bigdata &&
wget https://download.java.net/openjdk/jdk21/ri/openjdk-21+35_linux-x64_bin.tar.gz &&
tar -xzvf openjdk-21+35_linux-x64_bin.tar.g
```
---

### 3.3 Добавление JDBC-драйвера PostgreSQL
```bash
wget -P /opt/hive/lib https://jdbc.postgresql.org/download/postgresql-42.5.4.jar
```
---

### 3.4 Скопировать Hadoop конфиги
```bash
mkdir -p /opt/hive/conf &&
cp $HADOOP_CONF_DIR/core-site.xml /opt/hive/conf/ &&
cp $HADOOP_CONF_DIR/hdfs-site.xml /opt/hive/conf/
```
---

### 3.5 Настройка hive-site.xml
Создайте/отредактируйте `nano /opt/hive/conf/hive-site.xml`:
```xml
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://localhost:5432/hive_metastore</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>password</value>
  </property>
  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://0.0.0.0:9083</value>
  </property>
    <property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive</value>
  </property>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>
</configuration>
```
Замените пароли.

### 3.6 Добавьте следующие переменные в ``~/.bashrc`` и после выполните ``source ~/.bashrc``

```bash
# Setup HMS   
export HIVE_LOG_DIR=/var/log/hive
```
---

### 3.7 Инициализация схемы метаданных
Выполните schematool:
```bash
/opt/hive/bin/schematool -dbType postgres -initSchema
```

Если используете standalone JAR, используйте соответствующий скрипт/опцию для schematool.

---

### 3.8 Запуск Metastore (standalone)
Пример запуска в фоне:
```bash
nohup /opt/hive/bin/start-metastore --service metastore > /var/log/hive/metastore.log 2>&1 &
```
Systemd unit (рекомендуется) — `/etc/systemd/system/hive-metastore.service`:
```
[Unit]
Description=Hive Standalone Metastore
After=network.target remote-fs.target nss-lookup.target
Requires=network.target

[Service]
Type=simple
User=hdpuser
Group=hdpuser
Environment=HADOOP_HOME=/opt/hadoop
Environment=HIVE_HOME=/opt/hive
Environment=HIVE_CONF_DIR=/opt/hive/conf
Environment=JAVA_HOME=/opt/hadoop-jdk
Environment=PATH=/opt/hadoop/bin:/usr/bin:/bin
WorkingDirectory=/opt/hive
ExecStart=/bin/bash -lc /opt/hive/bin/start-metastore
ExecStop=/bin/bash -lc 'pkill -f HiveMetaStore || true'
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload &&
sudo systemctl enable --now hive-metastore
```
---

### 3.9 Подготовка HDFS warehouse
Убедитесь, что директория хранилища существует и имеет права:
```bash
hdfs dfs -mkdir -p /user/hive/warehouse &&
hdfs dfs -chown -R hdpuser:hdpuser /user/hive &&
hdfs dfs -chmod -R 0777 /user/hive &&
hdfs dfs -mkdir -p /external &&
hdfs dfs -chown -R hdpuser:hdpuser /external &&
hdfs dfs -chmod -R 0777 /external
```
(Адаптируйте владельца под вашу схему пользователей.)

- Проверить, что Metastore слушает порт 9083:
```bash
ss -ltnp | grep 9083
```
---