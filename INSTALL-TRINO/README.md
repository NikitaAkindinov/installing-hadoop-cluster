## Установка и настройка Trino (PrestoSQL) на трёх ВМ для работы с HDFS

Предполагается, что HDFS у вас уже развернут (NameNode + DataNode) на тех же трёх ВМ.

---

## 1. Требования и предположения
- OS: Linux (Debian/Ubuntu/CentOS) на всех трёх ВМ.  
- Java 24 установлена на всех узлах.  
- HDFS доступен по RPC и HTTP (адрес(а) NameNode известны).  
- Сеть между ВМ открыта (порт Trino 8080, порты HDFS/Metastore).  
- Будем запускать Trino: 1 coordinator + 2 worker (можно изменить).

---

## 2. Установка Trino и JAVA 24 на каждую ВМ
### 2.1 Создание пользователя (если необходимо, то изменять далее по тексту пользователя)
```bash
# создать пользователя и директории
sudo useradd -m -s /bin/bash -c "Trino service account" trino &&
sudo passwd trino &&
sudo usermod -aG adm trino
su - trino
```
### 2.2 Установка Java 24
- Откройте страницу с загрузкой дистрибутивов [JDK 24][java], скопируйте ссылку на дистрибутив и следуйте инструкциям по установке:

[java]: https://jdk.java.net/java-se-ri/24

```bash
cd ~/bigdata &&
wget https://github.com/adoptium/temurin24-binaries/releases/download/jdk-24.0.1%2B9/OpenJDK24U-jdk_x64_linux_hotspot_24.0.1_9.tar.gz &&
sudo tar -xzf OpenJDK24U-jdk_x64_linux_hotspot_24.0.1_9.tar.gz -C /opt &&
sudo rm  /opt/trino-java &&
sudo ln -s /opt/jdk-24.0.1+9 /opt/trino-java &&
sudo chown -R hdpuser:hdpuser /opt/trino-java* /opt/jdk-24* 
```
- Настройка переменных среды

```bash
nano ~/.bashrc  # добавьте следующее в конце файла

# User specific environment and startup programs
export PATH=$HOME/.local/bin:$HOME/bin:$PATH

# Setup JAVA Environment variables
export JAVA_HOME=/opt/trino-java
export PATH=$JAVA_HOME/bin:$PATH

source ~/.bashrc # загрузите .bashrc файл
```

### 2.3 Установка Trino
```bash
# пример (адаптировать версию)
wget https://repo1.maven.org/maven2/io/trino/trino-server/<VERSION>/trino-server-<VERSION>.tar.gz

cd /bigdata &&
wget http://nexsus.corp/repository/maven-public/io/trino/trino-server/476/trino-server-476.tar.gz &&
sudo tar -xzf trino-server-476.tar.gz -C /opt &&
sudo ln -s /opt/trino-server-476 /opt/trino &&
sudo mkdir -p /opt/trino/etc /opt/trino/etc/conf /var/lib/trino /var/log/trino &&
sudo chown -R hdpuser:hdpuser /opt/trino* /var/lib/trino /var/log/trino
```
---

## 3. Общие конфигурационные файлы
Создайте на каждом узле в /opt/trino/etc/ следующие файлы: `node.properties`, `jvm.config`, `config.properties` (с отличиями для coordinator/worker), `catalog/hive.properties`.

```bash
sudo nano /opt/trino/etc/node.properties
sudo nano /opt/trino/etc/jvm.config
```

node.properties (каждому узлу — уникальный node.id):
```
node.environment=production
node.id=<uuid> # сгенерировать
node.data-dir=/var/lib/trino
```

jvm.config (пример):
```
-Xmx8G
-XX:+UseG1GC
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
```

---

## 4. Настройка ролей: coordinator vs worker

```bash
sudo nano /opt/trino/etc/config.properties
```

Coordinator (выбираем одну ВМ):
```
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8080
query.max-memory=50GB
query.max-memory-per-node=1GB
discovery-server.enabled=true
discovery.uri=http://<COORDINATOR_HOST>:8080
```

Worker (на двух остальных):
```
coordinator=false
http-server.http.port=8080
discovery.uri=http://<COORDINATOR_HOST>:8080
query.max-memory-per-node=1GB
```

```
mkdir -p /opt/trino/etc/access-control &&
tee /opt/trino/etc/access-control/rules.json << EOF
{
  "catalogs": [
    {
      "catalog": "hive",
      "schemas": [
        {
          "schema": "*",
          "tables": [
            {
              "table": "*",
              "privileges": ["SELECT"]
            }
          ]
        }
      ]
    }
  ]
}
EOF

```

---

## 5. Коннектор Hive (чтобы Trino работал с HDFS) (проверить, работает ли без этого)
- Убедитесь, что плагин `hive` присутствует в /opt/trino/plugin (обычно входит в дистрибутив).
- Создайте файл `/opt/trino/etc/catalog/hive.properties`:
sudo mkdir /etc/trino/catalog

cd ~/ &&
wget http://nexsus.corp/repository/io/trino/trino-hive-hadoop2/476/trino-hive-hadoop2-476.tar.gz
sudo tar -xzf trino-hive-hadoop2-476.tar.gz -C trino-server-476/plugin/
https://repo1.maven.org/maven2/io/trino/trino-hive-hadoop2/434/trino-hive-hadoop2-434.zip

```bash
sudo cp $HADOOP_CONF_DIR/core-site.xml $HADOOP_CONF_DIR/hdfs-site.xml /opt/trino/etc/conf/ &&
sudo chown -R trino:trino /opt/trino/etc/conf/
nano /opt/trino/etc/catalog/hive.properties
```

```
connector.name=hive
hive.metastore.uri=thrift://<HIVE_METASTORE_HOST>:9083
hive.metastore.thrift.client.connect-timeout=10s
hive.metastore.thrift.client.read-timeout=10s
hive.metastore-cache-ttl=0s
#hive.security=sql-standard
#hive.force-local-scheduling=true
hive.non-managed-table-writes-enabled=true
hive.non-managed-table-creates-enabled=true
hive.fs.new-file-inherit-ownership=true
hive.dfs.replication=2

fs.hadoop.enabled=true
hive.fs.new-directory-permissions=0777

#hive.hadoop.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
#hive.warehouse.dir=hdfs://192.168.252.244:8020/user/hive/warehouse
#hive.security=file
#security.config-file=/opt/trino/etc/access-control/rules.json

```
Примечание:
- Если у вас нет Hive Metastore, установите его (MySQL/Postgres + HMS) — Trino требует метаданных для таблиц; напрямую "только HDFS" без метастора не работает нормально.

---

## 6. Hadoop-клиентские конфиги и права
- Скопируйте `core-site.xml` и `hdfs-site.xml` с Hadoop/HDFS на каждый узел Trino в `/opt/trino/etc/conf/`.
- Убедитесь, что `hive.config.resources` в hive.properties указывает на эти файлы.
- Проверьте права HDFS: пользователь `trino` (или тот, под кем запущен сервис) должен иметь необходимые права на пути в HDFS. При Kerberos — подготовьте keytab и principal.

---

## 7. Kerberos (опционально)
Если HDFS/Metastore используют Kerberos:
- Установите `krb5.conf` и разместите keytab для `trino` (например, `/opt/trino/etc/trino.keytab`).
- Добавьте JAAS/системные свойства в `jvm.config` и `config.properties` согласно документации Trino/Hive connector.
(Настройка Kerberos — отдельная подробная тема; при необходимости добавлю пример.)

---

## 8. Запуск Trino как systemd-сервис
Создайте unit `sudo nano /etc/systemd/system/trino.service`:
```
[Unit]
Description=Trino
After=network.target

[Service]
Type=forking
# Type=simple
User=hdpuser
Group=hdpuser
Environment="JAVA_HOME=/opt/java"
ExecStart=/opt/trino/bin/launcher start
#ExecStop=/opt/trino/bin/launcher stop
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Запустить:
```bash
sudo systemctl daemon-reload &&
sudo systemctl enable --now trino &&
sudo systemctl status trino.service

sudo journalctl -u trino.service -b --no-pager -n 200
```

---

## 9. Проверка статуса и базовая валидация
- Убедитесь, что worker зарегистрирован у coordinator:
```bash
curl http://<COORDINATOR_HOST>:8080/v1/node
```
- Подключение через cli:
```bash
# trino-cli (скачать trino-cli)
trino --server <COORDINATOR_HOST>:8080 --catalog hive --schema default
```
- Примеры команд:
```
SHOW CATALOGS;
SHOW SCHEMAS;
CREATE TABLE test_hdfs (id bigint, name varchar) WITH (format='ORC', external_location='hdfs://<NAMENODE>:8020/user/trino/test/');
INSERT INTO test_hdfs VALUES (1,'a');
SELECT * FROM test_hdfs;
```

---

## 10. Рекомендации по производительности
- Настроить `query.max-memory`, `query.max-memory-per-node` по доступной ОЗУ.  
- Подстроить количество потоков: `task.max-worker-threads`, `task.concurrency`.  
- Мониторить через Web UI (http://<COORDINATOR_HOST>:8080) и лог-файлы `/var/log/trino`.

---

## 11. Частые ошибки и проверки
- "Could not connect to metastore" — проверить `hive.metastore.uri` и сеть.  
- Ошибки прав HDFS — проверить права/ACL и user/keytab.  
- Несовместимые версии плагинов — используйте совместимую версию Trino и Hive connector.

---