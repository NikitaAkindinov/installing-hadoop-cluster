# Установка Hadoop как на одноузловой, так и на многоузловой кластер на основе виртуальных машин под управлением Debian 13 Linux

&nbsp;

> В этой статье описывается, как настроить кластер Hadoop на основе псевдораспределенной конфигурации. В первом разделе объясняется, как установить Hadoop в Debian 13 Linux. После этой установки кластер Hadoop будет состоять только из одного узла (т.е. кластера с одним узлом), и задания MapReduce будут выполняться псевдораспределенным образом. Чтобы использовать больше возможностей Hadoop, мы изменим конфигурацию, чтобы задания выполнялись распределенным образом, в итоге кластер Hadoop будет состоять не только из одного узла (т.е. многоузловой кластер).


> **Для успешного выполнения руководства необходимы как минимум виртуальные машины или физические сервера, оснащенные ОС Linux (Debian 13). В принципе, это должно работать даже с другими дистрибутивами Linux и другими версиями Debian. Вы можете использовать Virtualbox для создания виртуальных машин. Чтобы создать концепцию кластера, у вас также должна быть возможность создания более одной виртуальной машины.**

Существует несколько решений, к сожалению, как правило, платных, для создания виртуальных машин. Если вы предпочитаете бесплатные решения, я предлагаю вам создать свой кластер, установив бесплатные программные средства виртуализации на свой локальный компьютер. Очевидно, что для этого в дальнейшем потребуется достаточно ресурсов в виде памяти и места для хранения. Если вы предпочитаете это решение, выполните следующие действия:

- Скачайте [Oracle Virtualbox][virtualbox].
- Загрузите ОС [Debian][linux].
- [Сохдание виртуальной машины и установка ОС в нее][installdebian].
- [Клонирование ВМ][clone] after following the Hadoop installation steps.

[virtualbox]: https://www.virtualbox.org/wiki/Downloads
[linux]: https://www.debian.org/distrib/
[installdebian]: https://medium.com/platform-engineer/how-to-install-debian-linux-on-virtualbox-with-guest-additions-778afa0ee7e0
[clone]: https://protechgurus.com/clone-virtual-machine-virtualbox/

| :point_up:    | В следующем руководстве будет рассказано [как установить режимы Spark Standalone и Hadoop Yarn в многоузловом кластере][nexttuto]. |
|---------------|:------------------------|

[nexttuto]: https://github.com/mnassrib/installing-spark-standalone-and-hadoop-yarn-on-cluster

&nbsp;
&nbsp;

> # Установка и настройка Hadoop с NameNode и DataNode на одном узле

&nbsp;

## 1 - Подготовка ОС
### 1.1 Меняем имя сервера
---
| :warning: WARNING          |
|:---------------------------|
| Следующие команды необходимо запускать под пользователем с правами root     |

Измените имя хоста и настройте полное доменное имя (FQDN) (учитывая, что имя хоста и полное доменное имя являются `vm-hdp-0` и `corp` соответственно).
- Отобразите имя хоста, которое было задано во время настройки, на панели gridscale

```bash
cat /etc/hostname
```

- Измените имя хоста (теперь его можно изменить на любое другое имя – в данном примере vm-hdp-0).
```bash
nano /etc/hostname   # удалите существующее имя и напишите следующее
```

- Для того чтобы задать полное доменное имя, в дополнение к вашему собственному полному доменному имени требуется общедоступный IP-адрес сервера. Закомментируйте все имеющиеся строки, поставив перед ними символ `#` или убрав их и добавив следующую строку (заменив IP-адрес адресом сервера).

```bash
nano /etc/hosts

# ваш файл должен выглядеть следующим образом
192.168.252.253    vm-hdp-0.corp     vm-hdp-0
```
- Изменения вступят в силу после следующего перезапуска – если вы хотите, чтобы изменения были внесены без перезапуска, для этого выполните следующую команду ``hostname vm-hdp-0``

| :warning: WARNING          |
|:---------------------------|
| Изменение имени хоста с помощью команды `hostname vm-hdp-0` является временным и будет перезаписано при перезагрузке. Чтобы сделать это изменение постоянным, необходимо отредактировать файл `/etc/hostname`. Это не является заменой первого шага.     |

- Проверьте, что имя хоста было успешно отредактировано, набрав ``hostname`` должно вернуться
```
vm-hdp-0
```
- Полное доменное имя проверяется с помощью этой команды ``hostname -f`` должно вернуться
```
vm-hdp-0.corp
```

### 1.2 Монтирование диска для данных (если имеется)
---
- Если для данных добавлен дополнительный диск, то его необходимо отформатировать и примонтировать.
С помощью команды ``lsblk `` смотрим название нового устройства. Примерный вывод команды ниже:
>
    root@vm-hdp-0:~# lsblk 
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda      8:0    0   32G  0 disk 
    |-sda1   8:1    0 30.3G  0 part /
    |-sda2   8:2    0    1K  0 part 
    `-sda5   8:5    0  1.7G  0 part [SWAP]
    sdb      8:16   0  100G  0 disk 
    sr0     11:0    1 1024M  0 rom  

Далее необходимо создать раздел (пример для /dev/sdb)
Открыть fdisk: sudo fdisk /dev/sdb Команды внутри:
- n — новая раздел
- p — primary (по умолчанию)
- Enter для принятия значений начала/конца (весь диск)
- w — записать изменения и выйти

Получим такой вывод команды ``lsblk ``:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   32G  0 disk 
|-sda1   8:1    0 30.3G  0 part /
|-sda2   8:2    0    1K  0 part 
`-sda5   8:5    0  1.7G  0 part [SWAP]
sdb      8:16   0  100G  0 disk 
`-sdb1   8:17   0  100G  0 part 
sr0     11:0    1 1024M  0 rom  
```

- Далее необходимо отформатировать раздел с помощью команды ``mkfs.ext4 -F /dev/sdb1`` и выполнить следующие действия для монтирования диска.
```bash
sudo mkdir -p /data &&
sudo chown root:root /data &&
sudo chmod 755 /data &&
sudo blkid /dev/sdb1

echo 'UUID=4954a41d-b7ad-4596-96c9-8125a8ef2030  /data  ext4  defaults,noatime  0 2' | sudo tee -a /etc/fstab
sudo mount -a
df -hT /data
```
- Приверим, выполнив следующие команды: ``lsblk `` и ``df -l``:
```bash
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   32G  0 disk 
|-sda1   8:1    0 30.3G  0 part /
|-sda2   8:2    0    1K  0 part 
`-sda5   8:5    0  1.7G  0 part [SWAP]
sdb      8:16   0  100G  0 disk 
`-sdb1   8:17   0  100G  0 part /data
sr0     11:0    1 1024M  0 rom  

Filesystem     1K-blocks    Used Available Use% Mounted on
udev             4014888       0   4014888   0% /dev
tmpfs             807252     504    806748   1% /run
/dev/sda1       31113048 1017264  28490000   4% /
tmpfs            4036244       0   4036244   0% /dev/shm
tmpfs               5120       0      5120   0% /run/lock
tmpfs               1024       0      1024   0% /run/credentials/systemd-journald.service
tmpfs            4036248       0   4036248   0% /tmp
tmpfs               1024       0      1024   0% /run/credentials/getty@tty1.service
tmpfs             807248       8    807240   1% /run/user/0
/dev/sdb1      102625208    2072  97363924   1% /data
```
- Так же рекомендуется для удобства установить последние обновления ``apt update && apt upgrade -y`` и следующие программы 
``apt install -y mc htop``

### 1.3 Создать пользователя для Hadoop (рассматривая пользователя hadoop как "hdpuser")
---
- Для пользователей ОС Debian войдите в систему от имени пользователя root и выполните следующие действия:
```bash
apt update && apt install sudo -y
adduser hdpuser
usermod -aG sudo hdpuser  #чтобы добавить пользователя в группу sudo
getent group sudo  # чтобы проверить, был ли добавлен в группу новый пользователь Debian sudo
# чтобы удалить пользователя
deluser --remove-home username
```
> Более подробную информацию по правам смотрите [здесь][verifsudo].

[verifsudo]: https://phoenixnap.com/kb/create-a-sudo-user-on-debian

- Проверка доступа sudo в Debian
```bash
su - hdpuser  --перейдите в учетную запись пользователя, которую вы только что создали

``hdpuser@vm-hdp-0:~$ sudo whoami``  --запустите любую команду, для которой требуется доступ суперпользователя. Например, это должно указывать на то, что вы являетесь пользователем root.
```

![sudowhoami](/INSTALL-HDFS-YARN/images/sudowhoami.png)

- Альтернативный вариант это добавить пользователя Hadoop в файл sudoers (*), для получения более подробной информации смотрите это [ссылка][sudo].

[sudo]: https://www.geek17.com/fr/content/debian-9-stretch-installer-et-configurer-sudo-61
```bash
visudo -f /etc/sudoers  --и в нижеприведенном разделе добавьте

# User privilege specification
root    ALL=(ALL:ALL) ALL
hdpuser ALL=(ALL:ALL) ALL
```
### 1.4 Настройка SSH подключения
---

| :warning: WARNING          |
|:---------------------------|
| Следующие команды необходимо запускать под пользователем hdpuser     |

- Для дальнейших действий необходимо работать под пользователем hdpuser. Далее необходима установка SSH сервера и rsync, который позволяет удаленно синхронизировать файлы по SSH (если отсутствуют).
```bash
sudo apt install -y ssh rsync
# Сгенерируйте SSH-ключи и настройте SSH-соединение без пароля между службами Hadoop
ssh-keygen -t ed25519  # просто нажмите Enter для выбора всех вариантов
ssh-copy-id -i ~/.ssh/id_ed25519.pub hdpuser@vm-hdp-0  # (вы должны иметь возможность подключаться по ssh, не запрашивая пароль)
ssh hdpuser@vm-hdp-0
```
### 1.5 Настройка статического IP адреса на самой ВМ
---

- Для настройки статического IP необходимо произвести действия указанные ниже:
```bash
sudo nano /etc/network/interfaces
#### Добавьте это (заменить на свои IP адреса)
auto enp0s3
 iface enp0s3 inet static
      address 192.168.1.72
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
sudo systemctl restart networking # Применить изменения перезапустив сервис
ip addr show enp0s3 # Проверить новый IP адрес
sudo systemctl restart ssh # Перезапустить сервис SSH
ssh hdpuser@192.168.1.72 # Повторить SSH подключение
exit # Выйти из системы
ssh hdpuser@vm-hdp-0 # Повторить SSH подключение
exit # Выйти из системы
```

## 2 - Установка JDK и Hadoop
### 2.1 Установка Java
---
- Создание каталога для дистрибутивов:
```bash
sudo mkdir /bigdata &&
sudo chown -R hdpuser:hdpuser /bigdata &&
sudo chmod -R 770 /bigdata
```
- Откройте страницу с загрузкой дистрибутива [JDK 21][java], скопируйте ссылку на дистрибутив и следуйте инструкциям по установке:

[java]: https://jdk.java.net/java-se-ri/21

```bash
cd /bigdata &&
wget https://download.java.net/openjdk/jdk21/ri/openjdk-21+35_linux-x64_bin.tar.gz &&
tar -xzf /bigdata/openjdk-21+35_linux-x64_bin.tar.gz -C /opt &&
sudo ln -s /opt/jdk-21 /opt/hadoop-jdk &&
sudo chown -R $(whoami):$(whoami) /opt/jdk-21 /opt/hadoop-jdk
```

- Настройка переменных среды
```bash
nano ~/.bashrc
# добавьте следующее в конце файла

# User specific environment and startup programs
export PATH=$HOME/.local/bin:$HOME/bin:$PATH

# Setup JAVA Environment variables
export JAVA_HOME=/opt/hadoop-jdk
export PATH=$JAVA_HOME/bin:$PATH
export PATH=$HOME/.local/bin:$HOME/bin:$PATH
export PATH=$JAVA_HOME/bin:$PATH

# Для применения добавленных переменных выполните
source ~/.bashrc
```

- Проверка версии Java ``java -version``

```bash
hdpuser@vm-hdp-0:/bigdata$ java -version
openjdk version "21" 2023-09-19
OpenJDK Runtime Environment (build 21+35-2513)
OpenJDK 64-Bit Server VM (build 21+35-2513, mixed mode, sharing)
```

### 2.2 Установка Hadoop
---

- Загрузите файл дистрибутива с [Hadoop][hadoop] (скопируйте ссылку и вставьте в команду ``wget``), и следуйте инструкциям по установке:

[hadoop]: https://hadoop.apache.org/release.html

```bash
cd /bigdata &&
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.4.3/hadoop-3.4.3.tar.gz &&
tar -zxf /bigdata/hadoop-3.4.3.tar.gz -C /opt &&
sudo ln -s /opt/hadoop-3.4.3 /opt/hadoop &&
sudo mkdir -p /var/log/hadoop 
sudo chown -R $(whoami):$(whoami) /opt/hadoop-3.4.3 /opt/hadoop /var/log/hadoop &&
sudo chmod -R 770 /opt/hadoop-3.4.3 /opt/hadoop /var/log/hadoop
```


- Установите следующие переменные среды в ``~/.bashrc``  
```bash
# добавьте следующее после раздела "Переменные среды Java" в файл .bashrc
# Setup Hadoop Environment variables
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_NAMENODE_OPTS="-XX:+UseParallelGC"
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
#export HADOOP_YARN_HOME=$HADOOP_HOME
#export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
export HADOOP_LOG_DIR=/var/log/hadoop
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
export PATH=$HOME/.local/bin:$HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$PATH

export HADOOP_CLASSPATH=$HADOOP_CONF_DIR:$HADOOP_COMMON_HOME/*:$HADOOP_COMMON_HOME/lib/*:$HADOOP_HDFS_HOME/*:$HADOOP_HDFS_HOME/lib/*:$HADOOP_MAPRED_HOME/*:$HADOOP_MAPRED_HOME/lib/*:$HADOOP_YARN_HOME/*:$HADOOP_YARN_HOME/lib/*:$HADOOP_CLASSPATH

# Eсли планируете использовать только HDFS, то вставьте две следующих строки
# Control Hadoop
alias start_hadoop='$HADOOP_HOME/sbin/start-dfs.sh;'
alias stop_hadoop='$HADOOP_HOME/sbin/stop-dfs.sh;'

# Eсли планируете использовать HDFS, YARN, MR, то вставьте две следующих строки
# Control Hadoop
alias start_hadoop='$HADOOP_HOME/sbin/start-dfs.sh;start-yarn.sh;mapred --daemon start historyserver'
alias stop_hadoop='$HADOOP_HOME/sbin/stop-dfs.sh;stop-yarn.sh;mapred --daemon stop historyserver'
```

- После сохранения файла .bashrc, примените его командой ``source ~/.bashrc``

- Создайте каталог данных Hadoop для (NameNode и DataNode)

```bash
mkdir -p /bigdata/HadoopData/namenode # только на сервере NameNode
mkdir -p /data/HadoopData/datanode # на всех серверах DataNodes
sudo chown -R $(whoami):$(whoami) /data/HadoopData &&
sudo chmod -R 770 /data/HadoopData
```

- Конфигурирование Hadoop
```bash
cd $HADOOP_CONF_DIR  # перейдите в каталог с конфигурациями
nano core-site.xml # откройте файл для редактирования и вставьте конфигурацию указанную ниже
```

```xml
<configuration>
   <property>
       <name>fs.defaultFS</name>
       <value>hdfs://vm-hdp-0:9000</value>
   </property>
</configuration>
```

```bash
nano hdfs-site.xml # откройте файл для редактирования и вставьте конфигурацию указанную ниже
```

| :exclamation: | Параметр `dfs.namenode.data.dir` должен присутствовать только на сервере NameNode. Если на сервере NameNode необходима роль DataNode, то установите так же параметр `dfs.datanode.data.dir`.       |
|---------------|:------------------------|

```xml
<configuration>
   <property>
       <name>dfs.namenode.name.dir</name>
       <value>file:///bigdata/HadoopData/namenode</value>
   </property>
   <property>
       <name>dfs.datanode.data.dir</name>
       <value>file:///data/HadoopData/datanode</value>
   </property>
   <property>
       <name>dfs.blocksize</name>
       <value>134217728</value>
   </property>
   <property>
       <name>dfs.replication</name>
       <value>1</value>
   </property>
   <property>
       <name>dfs.permissions</name>
       <value>false</value>
   </property>
</configuration>
```

> Если в кластере необходимы MR вычисления
```bash
nano mapred-site.xml # откройте файл для редактирования и вставьте конфигурацию указанную ниже
```

```xml
<configuration>
   <property>
       <name>mapreduce.framework.name</name>
       <value>yarn</value>
   </property>
   <property>
       <name>mapreduce.jobhistory.address</name>
       <value>vm-hdp-0:10020</value>
   </property>
   <property>
       <name>mapreduce.jobhistory.webapp.address</name>
       <value>vm-hdp-0:19888</value>
   </property>
   <property>
       <name>mapreduce.jobhistory.intermediate-done-dir</name>
       <value>var/log/hadoop/tmp</value>
   </property>
   <property>
       <name>mapreduce.jobhistory.done-dir</name>
       <value>var/log/hadoop/done</value>
   </property>
   <property>
       <name>mapreduce.map.memory.mb</name>
       <value>512</value>
   </property>
   <property>
       <name>mapreduce.reduce.memory.mb</name>
       <value>512</value>
   </property>
   <property>
       <name>mapreduce.map.java.opts</name>
       <value>-Xmx512M</value>
   </property>
   <property>
       <name>mapreduce.job.maps</name>
       <value>2</value>
   </property>
   <property>
       <name>mapreduce.reduce.java.opts</name>
       <value>-Xmx512M</value>
   </property>
   <property>
       <name>mapreduce.task.io.sort.mb</name>
       <value>128</value>
   </property>
   <property>
       <name>mapreduce.task.io.sort.factor</name>
       <value>15</value>
   </property>
   <property>
       <name>mapreduce.reduce.shuffle.parallelcopies</name>
       <value>2</value>
   </property>
   <property>
       <name>yarn.app.mapreduce.am.env</name>
       <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
   </property>
   <property>
       <name>mapreduce.map.env</name>
       <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
   </property>
   <property>
       <name>mapreduce.reduce.env</name>
       <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
   </property>
   <property>
       <name>mapreduce.application.classpath</name>
       <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
   </property>
</configuration>
```

> Если в кластере необходим YARN
```bash
nano yarn-site.xml # откройте файл для редактирования и вставьте конфигурацию указанную ниже
```

```xml
<configuration>
   <property>
       <name>yarn.log-aggregation-enable</name>
       <value>true</value>
   </property>
   <property>
       <name>yarn.resourcemanager.address</name>
       <value>vm-hdp-0:8050</value>
   </property>
   <property>
       <name>yarn.resourcemanager.scheduler.address</name>
       <value>vm-hdp-0:8030</value>
   </property>
   <property>
       <name>yarn.resourcemanager.resource-tracker.address</name>
       <value>vm-hdp-0:8025</value>
   </property>
   <property>
       <name>yarn.resourcemanager.admin.address</name>
       <value>vm-hdp-0:8011</value>
   </property>
   <property>
       <name>yarn.resourcemanager.webapp.address</name>
       <value>vm-hdp-0:8080</value>
   </property>
   <property>
       <name>yarn.nodemanager.env-whitelist</name>
       <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
   </property>
   <property>
       <name>yarn.resourcemanager.webapp.https.address</name>
       <value>vm-hdp-0:8090</value>
   </property>
   <property>
       <name>yarn.resourcemanager.hostname</name>
       <value>vm-hdp-0</value>
   </property>
   <property>
       <name>yarn.resourcemanager.scheduler.class</name>
       <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
   </property>
   <property>
       <name>yarn.nodemanager.local-dirs</name>
       <value>file:///var/log/hadoop</value>
   </property>
   <property>
       <name>yarn.nodemanager.log-dirs</name>
       <value>file:///var/log/hadoop</value>
   </property>
   <property>
       <name>yarn.nodemanager.remote-app-log-dir</name>
       <value>hdfs://vm-hdp-0:9870/tmp/hadoop-yarn</value>
   </property>
   <property>
       <name>yarn.nodemanager.remote-app-log-dir-suffix</name>
       <value>logs</value>
   </property>
   <property>
       <name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
   </property>
   <property>
       <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
       <value>org.apache.hadoop.mapred.ShuffleHandler</value>
   </property>
   <property>
       <name>yarn.log.server.url</name>
       <value>http://vm-hdp-0:19888/jobhistory/logs</value>
   </property>
</configuration>
```

- Измените файл: **hadoop-env.sh** ``nano hadoop-env.sh``

| :memo:        | Отредактируйте файл среды Hadoop, добавив следующие переменные среды в разделе "Установите здесь переменные среды, специфичные для Hadoop".:       |
|---------------|:------------------------|

```bash
export JAVA_HOME=/opt/hadoop-jdk
export HADOOP_LOG_DIR=/var/log/hadoop
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/opt/hadoop/lib/native"
export HADOOP_COMMON_LIB_NATIVE_DIR=/opt/hadoop/lib/native
#export HDFS_DATANODE_OPTS="-Xms3g -Xmx6g" --для ограничения памяти для процессов
```


- Create **workers** file
```bash
nano $HADOOP_HOME/etc/hadoop/workers

#вставте содержимое ниже в файл. Запись localhost можно удалить

vm-hdp-0
```
- Форматирование нового хранилища HDFS в NameNode

```bash
hdfs namenode -format
```
- Запуск и остановка Hadoop

| :point_up:    | В принципе, чтобы запустить Hadoop, нам нужно всего лишь ввести `start-all.sh`. Однако ранее были созданы два псевдонима `start_hadoop` и `stop_hadoop` в переменных окружения, которые будут обеспечивать выполнение запуска и остановки Hadoop. Эти псевдонимы были созданы, чтобы избежать конфликтов с некоторыми командами, существующими в Spark, которые вскоре будут установлены на тех же серверах/ВМ. Те же правила будут применяться и в Spark. После завершения работы приложения вам необходимо запустить сервер истории заданий MapReduce, если вы хотите просмотреть журналы в веб-интерфейсе. Для этого были добавлены команды `mapred --daemon start historyserver` и `mapred --daemon stop historyserver` в два созданных псевдонима. |
|---------------|:------------------------|

###### Запуск
```bash
hdpuser@vm-hdp-0:~$ start_hadoop

Starting namenodes on [vm-hdp-0]
Starting datanodes
Starting secondary namenodes [vm-hdp-0]
Starting resourcemanager
Starting nodemanagers

###### Проверьте, запущены ли процессы Hadoop

hdpuser@vm-hdp-0:~$ jps  #эта команда должна возвращать что-то вроде

2961 Jps
2067 DataNode
2504 ResourceManager
1981 NameNode
2254 SecondaryNameNode
2591 NodeManager
```
###### Веб-интерфейсы по умолчанию

| Service   |      Address web      |  Default HTTP port |
|-----------|-----------------------|-------------------:|
| NameNode |  http://vm-hdp-0.corp:9870/ | 9870 |
| ResourceManager |    http://vm-hdp-0.corp:8080/   |   8080 |
| MapReduce JobHistory Server |    http://vm-hdp-0.corp:19888/   |   19888 |

###### Для примера можем загрузить файлы в HDFS
```bash
hdfs dfs -put /bigdata/openjdk-11.0.0.2_linux-x64.tar.gz /
```
###### Остановка
```bash
hdpuser@vm-hdp-0:~$ stop_hadoop
```
&nbsp;
&nbsp;

> # Установите Hadoop с NameNode и DataNode в многонодовом исполнении

&nbsp;

В втором разделе мы перейдем к созданию многоузлового кластера. Будут рассмотрены три виртуальные машины (узла). Если вы хотите создать кластер, состоящий из более чем трех узлов, вы можете применить те же шаги, которые будут описаны ниже. Подключение новых узлов будут происходить плавно, без остановки кластера с сохранением данных.

Предполагая, что имена хостов, ip-адреса и службы (NameNode и/или DataNode) трех узлов будут следующими:

| Hostname   |      IP Address     |  NameNode |  DataNode
|----------|-------------|:------:|:------:|
| vm-hdp-0 |  192.168.252.253 | &check; | &check; |
| vm-hdp-1 |  192.168.252.252   |   | &check; |
| vm-hdp-2 |  192.168.252.251   |   | &check; |

Пока что у нас готова только одна машина (vm-hdp-0). Нам нужно собрать и настроить две другие добавленные машины. Мы можем дважды клонировать машину vm-hdp-0, тогда необходимо будет именить только некоторые параметры.

## 1 - Дважды клонируйте сервер vm-hdp-0, созданный выше
### Команды с правами root
> войдите в систему как пользователь root на двух клонированных серверах

``hdpuser@vm-hdp-0:~$ su root``

- Измените имя хоста и настройте полное доменное имя (используя новые имена хостов как "vm-hdp-1" и "vm-hdp-2" и сохраняя то же полное доменное имя).

> На первой клонированной машине (vm-hdp-1)

``root@vm-hdp-0:~# nano /etc/hostname``  --удалите существующее имя и напишите следующее

    vm-hdp-1

``root@vm-hdp-0:~# nano /etc/hosts``  --ваш файл должен выглядеть следующим образом

    192.168.252.253    vm-hdp-0.corp    vm-hdp-0
    192.168.252.252    vm-hdp-1.corp    vm-hdp-1
    192.168.252.251    vm-hdp-2.corp    vm-hdp-2

``root@vm-hdp-0:~# hostname``  --должно вернуться

    vm-hdp-1

``root@vm-hdp-0:~# hostname -f``  --должно вернуться

    vm-hdp-1.corp

> На второй клонированной машине (vm-hdp-2 server)

``root@vm-hdp-0:~# nano /etc/hostname``  --удалите существующее имя и напишите следующее

    vm-hdp-2

``root@vm-hdp-0:~# nano /etc/hosts``  --ваш файл должен выглядеть следующим образом

    192.168.252.253    vm-hdp-0.corp    vm-hdp-0
    192.168.252.252    vm-hdp-1.corp    vm-hdp-1
    192.168.252.251    vm-hdp-2.corp    vm-hdp-2

``root@vm-hdp-0:~# hostname``  --должно вернуться

    vm-hdp-2

``root@vm-hdp-0:~# hostname -f``  --должно вернуться

    vm-hdp-2.corp



- Отредактируйте файл hosts на сервере "vm-hdp-0"

> войдите в систему как пользователь hdpuser на сервере "vm-hdp-0". Его файл hosts должен иметь то же содержимое, что и файлы hosts других узлов.

``hdpuser@vm-hdp-0:~$ sudo nano /etc/hosts``  --your file should look like the below

    192.168.252.253    vm-hdp-0.corp    vm-hdp-0
    192.168.252.252    vm-hdp-1.corp    vm-hdp-1
    192.168.252.251    vm-hdp-2.corp    vm-hdp-2

- Настройка SSH без пароля между службами Hadoop (с NN на DN)



### Настройка Hadoop
- Отредактируйте файл **workers** на сервере NameNode (vm-hdp-0)

| :warning: WARNING          |
|:---------------------------|
| Цель здесь состоит в том, чтобы настроить, в частности, рабочий файл в качестве NameNode или главного сервера (здесь это vm-hdp-0). Поскольку это позже организует работу всех серверов DataNode, он должен знать имена их узлов, указав их в своем рабочем файле. Этот файл является всего лишь вспомогательным файлом, который используется скриптами hadoop для запуска соответствующих служб на главном и подчиненных узлах. Добавьте рабочий файл только на главном узле (vm-hdp-0). Добавьте только имя или ip-адреса главного и всех подчиненных узлов. Если в файле есть запись для localhost, вы можете удалить ее. Что касается рабочих файлов серверов vm-hdp-1 и vm-hdp-2, отформатируйте их, оставив пустыми.     |

``hdpuser@vm-hdp-0:/bigdata/hadoop-3.4.1/etc/hadoop$ nano workers``  --строка записи для каждого сервера DataNode (в нашем случае все машины рассматриваются как DataNode)

    vm-hdp-0   #удалите эту строку из рабочего файла, если вы не хотите, чтобы этот узел был DataNode
    vm-hdp-1
    vm-hdp-2

- Измените файл: **hdfs-site.xml**

| :warning: WARNING          |
|:---------------------------|
| Если вам нужно, чтобы данные реплицировались более чем в один узел данных, вы должны изменить номер репликации, указанный в файлах **hdfs-site.xml** на всех узлах. Это число не может быть больше, чем количество узлов. Мы собираемся установить его равным 2. Это означает, что для каждого файла, хранящегося в HDFS, будет одна избыточная репликация этого файла на каком-либо другом узле кластера. А так же добавляем параметр dfs.hosts.exclude с указанием на файл dfs.exclude. У нем указываются ноды для вывода из эксплуатации или проведения долгого обслуживания    |

> На сервере NameNode & DataNode (vm-hdp-0): 

``hdpuser@vm-hdp-0:/bigdata/hadoop-3.4.1/etc/hadoop$ nano hdfs-site.xml``  --обновление конфигурации hdfs-site.xml

    <configuration>
       <property>
           <name>dfs.namenode.name.dir</name>
           <value>file:///bigdata/HadoopData/namenode</value>
       </property>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///data/HadoopData/datanode</value>
       </property>
       <property>
           <name>dfs.blocksize</name>
           <value>134217728</value>
       </property>
       <property>
           <name>dfs.replication</name>
           <value>2</value>
       </property>
       <property>
           <name>dfs.permissions</name>
           <value>false</value>
       </property>
       <property>
           <name>dfs.hosts.exclude</name>
           <value>/bigdata/hadoop-3.4.1/etc/hadoop/dfs.exclude</value>
       </property>
    </configuration>

``nano dfs.exclude`` --создание файла

> На серверах DataNode (vm-hdp-1 и vm-hdp-2):

``hdpuser@vm-hdp-1:/bigdata/hadoop-3.4.1/etc/hadoop$ nano hdfs-site.xml``  --обновление конфигурации hdfs-site.xml

    <configuration>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///data/HadoopData/datanode</value>
       </property>
       <property>
           <name>dfs.blocksize</name>
           <value>134217728</value>
       </property>
       <property>
           <name>dfs.replication</name>
           <value>2</value>
       </property>
       <property>
           <name>dfs.permissions</name>
           <value>false</value>
       </property>
    </configuration>

``hdpuser@vm-hdp-2:/bigdata/hadoop-3.4.1/etc/hadoop$ nano hdfs-site.xml``  --обновление конфигурации hdfs-site.xml

    <configuration>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///data/HadoopData/datanode</value>
       </property>
       <property>
           <name>dfs.blocksize</name>
           <value>134217728</value>
       </property>
       <property>
           <name>dfs.replication</name>
           <value>2</value>
       </property>
       <property>
           <name>dfs.permissions</name>
           <value>false</value>
       </property>
    </configuration>

- Очистите некоторые старые файлы на всех узлах

``hdpuser@vm-hdp-1:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@vm-hdp-2:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@vm-hdp-1:~$ rm -rf /data/HadoopData/datanode/*``

``hdpuser@vm-hdp-2:~$ rm -rf /data/HadoopData/datanode/*``


## 2- Запуск и остановка Hadoop на главном узле-namenode

- Запуск Hadoop

###### Запуск

``hdpuser@vm-hdp-0:~$ start_hadoop`` --При запуске будет ругаться, что часть процессов запущена. Но не запущенные запустятся.

![starthadoop](/INSTALL-HDFS-YARN/images/starthadoop.png)

###### Проверьте, запущены ли процессы Hadoop в vm-hdp-0

``hdpuser@vm-hdp-0:~$ jps``

![namenodejps](/INSTALL-HDFS-YARN/images/namenodejps.png)

###### Проверьте, запущены ли процессы Hadoop на vm-hdp-1

``hdpuser@mvm-hdp-1:~$ jps``

![datanode1jps](/INSTALL-HDFS-YARN/images/datanode1jps.png)

###### Проверьте, запущены ли процессы Hadoop на vm-hdp-2

``hdpuser@vm-hdp-2:~$ jps``

![datanode2jps](/INSTALL-HDFS-YARN/images/datanode2jps.png)

##### Но из-за того, что на vm-hdp-0 процесс NN и DN уже работали, то конфигурационный файл hdfs-site.xml не прочитался и изменения dfs.replication на 2 не применились. Для того, чтобы это исправить мы переведем DN на vm-hdp-0 сначала в Decommissioning (в этом состянии начнутся перераспределяться блоки на оставшиеся наши ноды), а далее нода vm-hdp-0 перейдет сама в режим Decommissioned( режим в котором DN процесс можно перезагрузить для применения конфигурации)

###### Веб-интерфейсы по умолчанию
> NameNode: http://vm-hdp-0.corp:9870/

![NameNode](/INSTALL-HDFS-YARN/images/master-node9870.png)

> ResourceManager: http://vm-hdp-0.corp:8080/

![ResourceManager](/INSTALL-HDFS-YARN/images/master-node8080.png)

###### Для перевода ноды в Decommissioning необходимо добавить название ноды vm-hdp-0 в файл dfs.exclude

``nano hadoop/dfs.exclude`` --добавляем в файл строку

    vm-hdp-0.corp

> После необходимо выполнить команду ``hdfs dfsadmin -refreshNodes`` NN читает файл и меняет статус у ноды. Статусы нод можно смотреть в веб интерфейсе или из отчета

###### Получить отчет

    hdpuser@vm-hdp-0:/bigdata/hadoop-3.4.1/etc/hadoop$ hdfs dfsadmin -report
    Configured Capacity: 210176425984 (195.74 GB)
    Present Capacity: 199401275392 (185.71 GB)
    DFS Remaining: 198231064576 (184.62 GB)
    DFS Used: 1170210816 (1.09 GB)
    DFS Used%: 0.59%
    Replicated Blocks:
    	Under replicated blocks: 0
    	Blocks with corrupt replicas: 0
    	Missing blocks: 0
    	Missing blocks (with replication factor 1): 0
    	Low redundancy blocks with highest priority to recover: 0
    	Pending deletion blocks: 0
    Erasure Coded Block Groups: 
    	Low redundancy block groups: 0
    	Block groups with corrupt internal blocks: 0
    	Missing block groups: 0
    	Low redundancy blocks with highest priority to recover: 0
    	Pending deletion blocks: 0

    -------------------------------------------------
    Live datanodes (3):

    Name: 192.168.252.251:9866 (vm-hdp-2.corp)
    Hostname: vm-hdp-2.corp
    Decommission Status : Normal
    Configured Capacity: 105088212992 (97.87 GB)
    DFS Used: 594341888 (566.81 MB)
    Non DFS Used: 2142208 (2.04 MB)
    DFS Remaining: 99106295808 (92.30 GB)
    DFS Used%: 0.57%
    DFS Remaining%: 94.31%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 0
    Last contact: Tue Mar 24 22:34:01 MSK 2026
    Last Block Report: Tue Mar 24 22:13:39 MSK 2026
    Num of Blocks: 5


    Name: 192.168.252.252:9866 (vm-hdp-1.corp)
    Hostname: vm-hdp-1.corp
    Decommission Status : Normal
    Configured Capacity: 105088212992 (97.87 GB)
    DFS Used: 575868928 (549.19 MB)
    Non DFS Used: 2142208 (2.04 MB)
    DFS Remaining: 99124768768 (92.32 GB)
    DFS Used%: 0.55%
    DFS Remaining%: 94.33%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 0
    Last contact: Tue Mar 24 22:34:02 MSK 2026
    Last Block Report: Tue Mar 24 22:26:26 MSK 2026
    Num of Blocks: 5


    Name: 192.168.252.253:9866 (vm-hdp-0.corp)
    Hostname: vm-hdp-0.corp
    Decommission Status : Decommissioned
    Configured Capacity: 105088212992 (97.87 GB)
    DFS Used: 1170206720 (1.09 GB)
    Non DFS Used: 2142208 (2.04 MB)
    DFS Remaining: 98530430976 (91.76 GB)
    DFS Used%: 1.11%
    DFS Remaining%: 93.76%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 0
    Last contact: Tue Mar 24 22:34:01 MSK 2026
    Last Block Report: Tue Mar 24 17:42:27 MSK 2026
    Num of Blocks: 10

###### Далее, когда мы убедились,что нода в статусе Decommissioned, необходимо остановить процесс DN на ноде с помощью команды ``hdfs --daemon stop datanode``. Проверяем статус ``hdfs --daemon status datanode`` или ``jps``.

    hdpuser@vm-hdp-0:/bigdata/hadoop-3.4.1/etc/hadoop$ hdfs --daemon status datanode
    datanode is stopped.
    hdpuser@vm-hdp-0:/bigdata/hadoop-3.4.1/etc/hadoop$ jps
    5457 NameNode
    3329 ResourceManager
    3077 SecondaryNameNode
    6009 NodeManager
    12490 Jps

###### Теперь можно запустить процесс ``hdfs --daemon start datanode``, удаляем ноду из файла dfs.exclude и обновляем информацию о нодах ``hdfs dfsadmin -refreshNodes`` и проверяем через отчет или в веб интерфейсе.

###### Но у уже имеющихся файлов будет реплива 1, когда по умолчанию для новых файлов будет 2. Чтобы это исправить, можно запустить команду ``hdfs dfs -setrep -R 2 /`` для изменения количества реплик для старых файлов. Для проверки работы этой задачи, можно использовать команду ``hdfs fsck /``, которая будет показывать блоки, для которых заплпнирована дополнительная репликация. Если необходимо ребалансировать блоки на нодах, то можно воспользоваться командой ``hdfs balancer -threshold 1``. Где 1 это порог перераспределения данных (в процентах от общей ёмкости диска) при запуске hdfs balancer. 

###### Остановка Hadoop

``hdpuser@vm-hdp-0:~$ stop_hadoop``

![stophadoop](/INSTALL-HDFS-YARN/images/stophadoop.png)

&nbsp;

| :point_up:    | В следующем руководстве объясняется [как установить сервисы Spark Standalone и Hadoop Yarn в многоузловом кластере][nexttuto]. |
|---------------|:------------------------|

[nexttuto]: https://github.com/mnassrib/installing-spark-standalone-and-hadoop-yarn-on-cluster
