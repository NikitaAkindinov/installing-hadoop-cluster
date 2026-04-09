# Обновление HDFS до hadoop-3.4.3
## 1. Загрузка и распаковка необходимых дистрибутивов

```bash
cd /bigdata &&
wget http://nexsus.corp/repository/archive.apache.org/dist/hadoop/common/hadoop-3.4.3/hadoop-3.4.3.tar.gz &&
wget https://download.java.net/openjdk/jdk21/ri/openjdk-21+35_linux-x64_bin.tar.gz &&
tar -zxvf hadoop-3.4.3.tar.gz && tar -xzvf openjdk-21+35_linux-x64_bin.tar.gz

```
## 2. Переносим конфигурации и редактируем в них пути

```bash
cd /bigdata/hadoop-3.4.1/etc/hadoop/ &&
scp core-site.xml hadoop-env.sh hdfs-site.xml /bigdata/hadoop-3.4.3/etc/hadoop/ &&
nano /bigdata/hadoop-3.4.3/etc/hadoop/hadoop-env.sh &&
cd /bigdata/hadoop-3.4.3/etc/hadoop/ &&
scp core-site.xml hadoop-env.sh hdfs-site.xml vm-hdp-bi-2:/bigdata/hadoop-3.4.3/etc/hadoop/
```
## 3. Остановка кластера HDFS

``stop_hadoop``

## 4. Делаем резервную копию каталога dfs.namenode.name.dir (fsimage, edits)

```bash
scp -r /bigdata/HadoopData/namenode/ /bigdata/HadoopData/namenodeBKP/
```

## 5. Редактируем переменные окружения пользователя и распространяем на другие ноды

```bash
nano ~/.bashrc 
scp ~/.bashrc vm-hdp-bi-1:~/.bashrc
# Выполнить на всех нодах
source ~/.bashrc
```

# 6. Пробуем запуститься и проверяем работоспособность

> Запускаем через start_hadoop. Проверяем в логах NN ``STARTUP_MSG:   version = 3.4.3`` и ``STARTUP_MSG:   java = 21``