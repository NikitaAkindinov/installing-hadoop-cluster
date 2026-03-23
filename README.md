# Установка Hadoop как на одноузловой, так и на многоузловой кластер на основе виртуальных машин под управлением Debian 9 Linux

&nbsp;

> В этой статье описывается, как настроить кластер Hadoop на основе псевдораспределенной конфигурации. В первом разделе объясняется, как установить Hadoop в Debian 9 Linux. После этой установки кластер Hadoop будет состоять только из одного узла (т.е. кластера с одним узлом), и задания MapReduce будут выполняться псевдораспределенным образом. Чтобы использовать больше возможностей Hadoop, мы изменим конфигурацию, чтобы задания выполнялись распределенным образом, в итоге кластер Hadoop будет состоять не только из одного узла (т.е. многоузловой кластер).


> **Для успешного выполнения руководства необходимы как минимум виртуальные машины или физические сервера, оснащенные ОС Linux (Debian 9). В принципе, это должно работать даже с другими дистрибутивами Linux. Вы можете использовать Virtualbox для создания виртуальных машин. Чтобы создать концепцию кластера, у вас также должна быть возможность создания более одной виртуальной машины.**

Существует несколько решений, к сожалению, как правило, платных, для приобретения машин для создания кластера. Если вы предпочитаете бесплатные решения, я предлагаю вам создать свой кластер, установив виртуальные машины Linux на свой локальный компьютер. Очевидно, что для этого в дальнейшем потребуется достаточно ресурсов в виде памяти и места для хранения. Если вы предпочитаете это решение, выполните следующие действия:

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

## 1- Подготовка ОС
### Команды с правами root
> login as root user

``user@debian:~$ su root``

- Turnoff firewall <span style="color: red;">(сомнительно, что это необходимо)</span>

``root@debian:~# service firewalld status``

``root@debian:~# service firewalld stop``

``root@debian:~# systemctl disable firewalld``

- Измените имя хоста и настройте полное доменное имя (FQDN) (учитывая, что имя хоста и полное доменное имя являются `master-namenode` и `cluster.hdp` соответственно).

> Отобразите имя хоста, которое было задано во время настройки, на панели gridscale

``root@debian:~# cat /etc/hostname``

> Измените имя хоста (теперь его можно изменить на любое другое имя – в данном примере master-namenode).

``root@debian:~# vi /etc/hostname``   --удалите существующее имя и напишите следующее

    master-namenode

> Для того чтобы задать полное доменное имя, в дополнение к вашему собственному полному доменному имени требуется общедоступный IP-адрес сервера. Закомментируйте все имеющиеся строки, поставив перед ними символ `#` или убрав их и добавив следующую строку (заменив IP-адрес адресом сервера).

``root@debian:~# vi /etc/hosts``   --ваш файл должен выглядеть следующим образом

    192.168.1.72    master-namenode.cluster.hdp     master-namenode

> Изменения вступят в силу после следующего перезапуска – если вы хотите, чтобы изменения были внесены без перезапуска, для этого выполните следующую команду

``root@debian:~# hostname master-namenode``

| :warning: WARNING          |
|:---------------------------|
| Изменение имени хоста с помощью команды `:~# hostname master-namenode` является временным и будет перезаписано при перезагрузке. Чтобы сделать это изменение постоянным, необходимо отредактировать файл `/etc/hostname`. Это не является заменой первого шага.     |

> Проверьте, что имя хоста было успешно отредактировано, набрав

``root@debian:~# hostname`` --должно вернуться

    master-namenode

> Полное доменное имя проверяется с помощью этой команды

``root@debian:~# hostname -f`` --should return

    master-namenode.cluster.hdp

- Создать пользователя для Hadoop (рассматривая пользователя hadoop как "hdpuser")
> Для пользователей ОС Debian войдите в систему от имени пользователя root и выполните следующие действия:

``root@master-namenode:~# apt-get install sudo``

``root@master-namenode:~# adduser hdpuser``

``root@master-namenode:~# usermod -aG sudo hdpuser``  --чтобы добавить пользователя в группу sudo. Это также можно сделать в соответствии с (*), приведенным ниже

``root@master-namenode:~# getent group sudo``  --чтобы проверить, был ли добавлен в группу новый пользователь Debian sudo, более подробную информацию смотрите [здесь][verifsudo].

[verifsudo]: https://phoenixnap.com/kb/create-a-sudo-user-on-debian

``root@master-namenode:~# deluser --remove-home username`` --чтобы удалить пользователя

> Проверка доступа Sudo в Debian

``root@master-namenode:~# su - hdpuser``  --перейдите в учетную запись пользователя, которую вы только что создали

``hdpuser@master-namenode:~$ sudo whoami``  --запустите любую команду, для которой требуется доступ суперпользователя. Например, это должно указывать на то, что вы являетесь пользователем root.

![sudowhoami](/images/sudowhoami.png)

- Добавьте пользователя Hadoop в файл sudoers (*), для получения более подробной информации смотрите это [ссылка][sudo].

[sudo]: https://www.geek17.com/fr/content/debian-9-stretch-installer-et-configurer-sudo-61

``root@master-namenode:~# visudo -f /etc/sudoers``  --и в нижеприведенном разделе добавьте

    ## Allow root to run any commands anywhere
    root    ALL=(ALL)    All
    hdpuser ALL=(ALL)    ALL     ##add this line

### Команды от hdpuser
> login as hdpuser

- Установка SSH сервера (если отсутствует)

``hdpuser@master-namenode:~$ sudo apt-get install ssh``

- Установите rsync, который позволяет удаленно синхронизировать файлы по SSH

``hdpuser@master-namenode:~$ sudo apt-get install rsync``

- Сгенерируйте SSH-ключи и настройте SSH-соединение без пароля между службами Hadoop

``hdpuser@master-namenode:~$ ssh-keygen -t rsa``  --просто нажмите Enter для выбора всех вариантов

``hdpuser@master-namenode:~$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys``

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-namenode``  --(вы должны иметь возможность подключаться по ssh, не запрашивая пароль)

``hdpuser@master-namenode:~$ ssh hdpuser@master-namenode``

    Are you sure you want to continue connecting (yes/no)? yes

> Если на этом этапе у вас возникли проблемы, это связано с тем, что ваша система использует другой инструмент настройки сети (скорее всего, ``/etc/network/interfaces`` в старых системах Debian). Вот как решить эти проблемы:
  1) Откройте конфигурационный файл: ``sudo nano /etc/network/interfaces``
  2) #### Добавьте это (заменить на свои IP адреса)
    auto enp0s3
     iface enp0s3 inet static
          address 192.168.1.72
        netmask 255.255.255.0
        gateway 192.168.1.1
        dns-nameservers 8.8.8.8 8.8.4.4
  3) Применить изменения перезапустив сервис: ``sudo systemctl restart networking``
  4) Проверить новый IP адрес: ``ip addr show enp0s3``
  5) Перезапустить сервис SSH: ``sudo systemctl restart ssh``
  6) Повторить SSH подключение: ``ssh hdpuser@192.168.1.72`` or ``ssh hdpuser@master-namenode``
  7) Выйти из системы ``hdpuser@master-namenode:~$ exit``

- Создание необходимых каталогов:

``hdpuser@master-namenode:~$sudo mkdir /var/log/hadoop``

``hdpuser@master-namenode:~$ sudo chown -R hdpuser:hdpuser /var/log/hadoop``

``hdpuser@master-namenode:~$ sudo chmod -R 770 /var/log/hadoop``

``hdpuser@master-namenode:~$ sudo mkdir /bigdata``

``hdpuser@master-namenode:~$ sudo chown -R hdpuser:hdpuser /bigdata``

``hdpuser@master-namenode:~$ sudo chmod -R 770 /bigdata``

## 2- Установка JDK и Hadoop
> Ввойти в систему под пользователем hdpuser

### Установка Java

- Загрузка JDK "[jdk-8u241-Linux-x64.tar.gz][java]", и следуйте инструкциям по установке:

[java]: https://www.oracle.com/java/technologies/javase-jdk8-downloads.html

``hdpuser@master-namenode:~$ cd /bigdata``

- Распакуйте архив по пути установки,

``hdpuser@master-namenode:/bigdata$ tar -xzvf jdk-8u241-Linux-x64.tar.gz``

- Настройка переменных среды

``hdpuser@master-namenode:/bigdata$ cd ~``

``hdpuser@master-namenode:~$ vi .bashrc``  --добавьте следующее в конце файла

    # User specific environment and startup programs
    export PATH=$HOME/.local/bin:$HOME/bin:$PATH

    # Setup JAVA Environment variables
    export JAVA_HOME=/bigdata/jdk1.8.0_241
    export PATH=$JAVA_HOME/bin:$PATH

``hdpuser@master-namenode:~$ source .bashrc`` --загрузите .bashrc файл

- Установка Java

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/java" "java" "/bigdata/jdk1.8.0_241/bin/java" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/javac" "javac" "/bigdata/jdk1.8.0_241/bin/javac" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/bigdata/jdk1.8.0_241/bin/javaws" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --set java /bigdata/jdk1.8.0_241/bin/java``

``hdpuser@master-namenode:~$ sudo update-alternatives --set javac /bigdata/jdk1.8.0_241/bin/javac``

``hdpuser@master-namenode:~$ sudo update-alternatives --set javaws /bigdata/jdk1.8.0_241/bin/javaws``

``hdpuser@master-namenode:~$ java -version``  --проверка версии

    hdpuser@master-namenode:~$ java -version
    java version "1.8.0_241"
    Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
    Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)


### Установка Hadoop

- Загрузите файл дистрибутива с Hadoop "[hadoop-3.1.2.tar.gz][hadoop]", и следуйте инструкциям по установке:

[hadoop]: https://hadoop.apache.org/release.html

``hdpuser@master-namenode:~$ cd /bigdata``

- Извлеките архив "hadoop-3.1.2.tar.gz",

``hdpuser@master-namenode:/bigdata$ tar -zxvf hadoop-3.1.2.tar.gz``

- Установите следующие переменные среды

``hdpuser@master-namenode:/bigdata$ cd``  --чтобы перейти в свой домашний каталог

``hdpuser@master-namenode:~$ vi .bashrc``  --добавьте следующее после раздела "Переменные среды Java" в файл .bashrc

    # Setup Hadoop Environment variables
    export HADOOP_HOME=/bigdata/hadoop-3.1.2
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
    export HADOOP_NAMENODE_OPTS="-XX:+UseParallelGC"
    export HADOOP_MAPRED_HOME=$HADOOP_HOME
    export HADOOP_HDFS_HOME=$HADOOP_HOME
    export HADOOP_COMMON_HOME=$HADOOP_HOME
    export HADOOP_YARN_HOME=$HADOOP_HOME
    export YARN_HOME=$HADOOP_HOME
    export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
    export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
    export HADOOP_LOG_DIR=/var/log/hadoop
    export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
    export PATH=$HOME/.local/bin:$HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$PATH

    export HADOOP_CLASSPATH=$HADOOP_CONF_DIR:$HADOOP_COMMON_HOME/*:$HADOOP_COMMON_HOME/lib/*:$HADOOP_HDFS_HOME/*:$HADOOP_HDFS_HOME/lib/*:$HADOOP_MAPRED_HOME/*:$HADOOP_MAPRED_HOME/lib/*:$HADOOP_YARN_HOME/*:$HADOOP_YARN_HOME/lib/*:$HADOOP_CLASSPATH

    # Control Hadoop
    alias Start_HADOOP='$HADOOP_HOME/sbin/start-dfs.sh;start-yarn.sh;mapred --daemon start historyserver'
    alias Stop_HADOOP='$HADOOP_HOME/sbin/stop-dfs.sh;stop-yarn.sh;mapred --daemon stop historyserver'

``hdpuser@master-namenode:~$ source .bashrc`` --после сохранения файла .bashrc, загрузите его

- Создайте каталог данных Hadoop для (NameNode и DataNode)

``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData``

``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData/namenode``      --*только на сервере NameNode*

``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData/datanode``      --*на всех серверах DataNodes*

- Конфигурирование Hadoop

``hdpuser@master-namenode:~$ cd $HADOOP_CONF_DIR``  --проверьте переменные среды, которые вы только что добавили

- Измените файл: **core-site.xml**

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi core-site.xml``  --copy core-site.xml file <span style="color: red;">Возможно имеется в виду создать файл и скопировать в него это</span>

    <configuration>
       <property>
           <name>fs.defaultFS</name>
           <value>hdfs://master-namenode:9000</value>
       </property>
    </configuration>

- Измените файл: **hdfs-site.xml**

| :exclamation: | Параметр `dfs.namenode.data.dir` должен присутствовать только на сервере NameNode. Если на сервере NameNode необходима роль DataNode, то установите так же параметр `dfs.datanode.data.dir`.       |
|---------------|:------------------------|

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file <span style="color: red;">Возможно имеется в виду создать файл и скопировать в него это</span>

    <configuration>
       <property>
           <name>dfs.namenode.name.dir</name>
           <value>file:///bigdata/HadoopData/namenode</value>
       </property>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///bigdata/HadoopData/datanode</value>
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

- Modify file: **mapred-site.xml**

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi mapred-site.xml``  --copy mapred-site.xml file <span style="color: red;">Возможно имеется в виду создать файл и скопировать в него это</span>

    <configuration>
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
       <property>
           <name>mapreduce.jobhistory.address</name>
           <value>master-namenode:10020</value>
       </property>
       <property>
           <name>mapreduce.jobhistory.webapp.address</name>
           <value>master-namenode:19888</value>
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
           <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
       </property>
       <property>
           <name>mapreduce.map.env</name>
           <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
       </property>
       <property>
           <name>mapreduce.reduce.env</name>
           <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
       </property>
       <property>
           <name>mapreduce.application.classpath</name>
           <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
       </property>
    </configuration>

- Modify file: **yarn-site.xml** <span style="color: red;">Возможно имеется в виду создать файл и скопировать в него это</span>

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi yarn-site.xml``  --copy yarn-site.xml file

    <configuration>
       <property>
           <name>yarn.log-aggregation-enable</name>
           <value>true</value>
       </property>
       <property>
           <name>yarn.resourcemanager.address</name>
           <value>master-namenode:8050</value>
       </property>
       <property>
           <name>yarn.resourcemanager.scheduler.address</name>
           <value>master-namenode:8030</value>
       </property>
       <property>
           <name>yarn.resourcemanager.resource-tracker.address</name>
           <value>master-namenode:8025</value>
       </property>
       <property>
           <name>yarn.resourcemanager.admin.address</name>
           <value>master-namenode:8011</value>
       </property>
       <property>
           <name>yarn.resourcemanager.webapp.address</name>
           <value>master-namenode:8080</value>
       </property>
       <property>
           <name>yarn.nodemanager.env-whitelist</name>
           <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
       </property>
       <property>
           <name>yarn.resourcemanager.webapp.https.address</name>
           <value>master-namenode:8090</value>
       </property>
       <property>
           <name>yarn.resourcemanager.hostname</name>
           <value>master-namenode</value>
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
           <value>hdfs://master-namenode:9870/tmp/hadoop-yarn</value>
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
           <value>http://master-namenode:19888/jobhistory/logs</value>
       </property>
    </configuration>

- Modify file: **hadoop-env.sh** <span style="color: red;">Возможно имеется в виду создать файл и скопировать в него это</span>

| :memo:        | Отредактируйте файл среды Hadoop, добавив следующие переменные среды в разделе "Установите здесь переменные среды, специфичные для Hadoop".:       |
|---------------|:------------------------|

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hadoop-env.sh``  --copy hadoop-env.sh

    export JAVA_HOME=/bigdata/jdk1.8.0_241
    export HADOOP_LOG_DIR=/var/log/hadoop
    export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/bigdata/hadoop-3.1.2/lib/native"
    export HADOOP_COMMON_LIB_NATIVE_DIR=/bigdata/hadoop-3.1.2/lib/native

- Create **workers** file

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi workers``  --copy workers file
| :exclamation: | Строка записи для каждого сервера DataNode       |
|---------------|:------------------------|

    master-namenode

- Форматирование нового хранилища HDFS в NameNode

``hdpuser@master-namenode:~$ hdfs namenode -format``

- Запуск и остановка Hadoop

| :point_up:    | В принципе, чтобы запустить Hadoop, нам нужно всего лишь ввести `start-all.sh`. Однако ранее были созданы два псевдонима `Start_HADOOP` и `Stop_HADOOP` в переменных окружения, которые будут обеспечивать выполнение запуска и остановки Hadoop. Эти псевдонимы были созданы, чтобы избежать конфликтов с некоторыми командами, существующими в Spark, которые вскоре будут установлены на тех же серверах/ВМ. Те же правила будут применяться и в Spark. После завершения работы приложения вам необходимо запустить сервер истории заданий MapReduce, если вы хотите просмотреть журналы в веб-интерфейсе. Для этого были добавлены команды `mapred --daemon start historyserver` и `mapred --daemon stop historyserver` в два созданных псевдонима. |
|---------------|:------------------------|

###### Запуск

``hdpuser@master-namenode:~$ Start_HADOOP``

###### Проверьте, запущены ли процессы Hadoop

    hdpuser@master-namenode:~$ jps  --эта команда должна возвращать что-то вроде
    1889 ResourceManager
    1300 NameNode
    1093 JobHistoryServer
    1993 NodeManager
    2426 Jps
    1403 DataNode
    1566 SecondaryNameNode

###### Веб-интерфейсы по умолчанию

| Service   |      Address web      |  Default HTTP port |
|-----------|-----------------------|-------------------:|
| NameNode |  http://master-namenode:9870/ | 9870 |
| ResourceManager |    http://master-namenode:8080/   |   8080 |
| MapReduce JobHistory Server |    http://master-namenode:19888/   |   19888 |

###### Остановка

``hdpuser@master-namenode:~$ Stop_HADOOP``

&nbsp;
&nbsp;

> # Установите Hadoop с NameNode и DataNode в многонодовом исполнении

&nbsp;

В втором разделе мы перейдем к созданию многоузлового кластера. Будут рассмотрены три виртуальные машины (узла). Если вы хотите создать кластер, состоящий из более чем трех узлов, вы можете применить те же шаги, которые будут описаны ниже.

Предполагая, что имена хостов, ip-адреса и службы (NameNode и/или DataNode) трех узлов будут следующими:

| Hostname   |      IP Address     |  NameNode |  DataNode
|----------|-------------|:------:|:------:|
| master-namenode |  192.168.1.72 | &check; | &check; |
| slave-datanode-1 |    192.168.1.73   |   | &check; |
| slave-datanode-2 |    192.168.1.74   |   | &check; |

Пока что у нас готова только одна машина (master-namenode). Нам нужно собрать и настроить две другие добавленные машины. Мы можем дважды клонировать машину master-namenode, тогда необходимо будет именить только некоторые параметры.

## 1- Дважды клонируйте сервер master-namenode, созданный выше
### Команды с правами root
> войдите в систему как пользователь root на двух клонированных серверах

``hdpuser@master-namenode:~$ su root``

- Измените имя хоста и настройте полное доменное имя (используя новые имена хостов как "slave-datanode-1" и "slave-datanode-2" и сохраняя то же полное доменное имя).

> На первой клонированной машине (slave-datanode-1)

``root@master-namenode:~# vi /etc/hostname``  --удалите существующее имя и напишите следующее

    slave-datanode-1

``root@master-namenode:~# vi /etc/hosts``  --ваш файл должен выглядеть следующим образом

    192.168.1.72    master-namenode.cluster.hdp     master-namenode
    192.168.1.73    slave-datanode-1.cluster.hdp    slave-datanode-1
    192.168.1.74    slave-datanode-2.cluster.hdp    slave-datanode-2

``root@master-namenode:~# hostname slave-datanode-1``

``root@master-namenode:~# hostname``  --должно вернуться

    slave-datanode-1

``root@master-namenode:~# hostname -f``  --должно вернуться

    slave-datanode-1.cluster.hdp

> На второй клонированной машине (slave-datanode-2 server)

``root@master-namenode:~# vi /etc/hostname``  --удалите существующее имя и напишите следующее

    slave-datanode-2

``root@master-namenode:~# vi /etc/hosts``  --ваш файл должен выглядеть следующим образом

    192.168.1.72    master-namenode.cluster.hdp     master-namenode
    192.168.1.73    slave-datanode-1.cluster.hdp    slave-datanode-1
    192.168.1.74    slave-datanode-2.cluster.hdp    slave-datanode-2

``root@master-namenode:~# hostname slave-datanode-2``

``root@master-namenode:~# hostname``  --должно вернуться

    slave-datanode-2

``root@master-namenode:~# hostname -f``  --должно вернуться

    slave-datanode-2.cluster.hdp



- Отредактируйте файл hosts на сервере "master-namenode"

> войдите в систему как пользователь hdpuser на сервере "master-namenode". Его файл hosts должен иметь то же содержимое, что и файлы hosts других узлов.

``hdpuser@master-namenode:~$ sudo vi /etc/hosts``  --your file should look like the below

    192.168.1.72    master-namenode.cluster.hdp     master-namenode
    192.168.1.73    slave-datanode-1.cluster.hdp    slave-datanode-1
    192.168.1.74    slave-datanode-2.cluster.hdp    slave-datanode-2

- Настройка SSH без пароля между службами Hadoop

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@slave-datanode-1``

``hdpuser@master-namenode:~$ ssh hdpuser@slave-datanode-1``

    Are you sure you want to continue connecting (yes/no)? yes

``hdpuser@slave-datanode-1:~$ exit``

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@slave-datanode-2``

``hdpuser@master-namenode:~$ ssh hdpuser@slave-datanode-2``

    Are you sure you want to continue connecting (yes/no)? yes

``hdpuser@slave-datanode-2:~$ exit``

### Настройка Hadoop
- Отредактируйте файл **workers** на сервере NameNode (master-namenode)

| :warning: WARNING          |
|:---------------------------|
| Цель здесь состоит в том, чтобы настроить, в частности, рабочий файл в качестве NameNode или главного сервера (здесь это master-namenode). Поскольку это позже организует работу всех серверов DataNode, он должен знать имена их узлов, указав их в своем рабочем файле. Этот файл является всего лишь вспомогательным файлом, который используется скриптами hadoop для запуска соответствующих служб на главном и подчиненных узлах. Добавьте рабочий файл только на главном узле (master-namenode). Добавьте только имя или ip-адреса главного и всех подчиненных узлов. Если в файле есть запись для localhost, вы можете удалить ее. Что касается рабочих файлов серверов slave-datanode-1 и slave-datanode-2, отформатируйте их, оставив пустыми.     |

``hdpuser@master-namenode:~$ vi workers``  --строка записи для каждого сервера DataNode (в нашем случае все машины рассматриваются как DataNode)

    master-namenode   #удалите эту строку из рабочего файла, если вы не хотите, чтобы этот узел был DataNode
    slave-datanode-1
    slave-datanode-2

- Измените файл: **hdfs-site.xml**

| :warning: WARNING          |
|:---------------------------|
| Если вам нужно, чтобы данные реплицировались более чем в один узел данных, вы должны изменить номер репликации, указанный в файлах **hdfs-site.xml** на всех узлах. Это число не может быть больше, чем количество узлов. Мы собираемся установить его равным 2. Это означает, что для каждого файла, хранящегося в HDFS, будет одна избыточная репликация этого файла на каком-либо другом узле кластера.    |

> На сервере NameNode & DataNode (master-namenode):

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

    <configuration>
       <property>
           <name>dfs.namenode.name.dir</name>
           <value>file:///bigdata/HadoopData/namenode</value>
       </property>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///bigdata/HadoopData/datanode</value>
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

> На серверах DataNode (slave-datanode-1 и slave-datanode-2):

``hdpuser@slave-datanode-1:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

    <configuration>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///bigdata/HadoopData/datanode</value>
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

``hdpuser@slave-datanode-2:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

    <configuration>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:///bigdata/HadoopData/datanode</value>
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

- Очистите некоторые старые файлы на всех узлах <span style="color: red;">Проверить/попробовать сделать через добавление новых нод. А не через создание нового кластера!</span>

``hdpuser@master-namenode:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@master-namenode:~$ rm -rf /bigdata/HadoopData/datanode/*``

``hdpuser@slave-datanode-1:~$ rm -rf /bigdata/HadoopData/datanode/*``

``hdpuser@slave-datanode-2:~$ rm -rf /bigdata/HadoopData/datanode/*``


## 2- Запуск и остановка Hadoop на главном узле-namenode

- Форматирование нового каталога HDFS на NameNode

``hdpuser@master-namenode:~$ hdfs namenode -format``

![format1](/images/format1.png)
![format2](/images/format2.png)

- Запуск Hadoop

###### Запуск

``hdpuser@master-namenode:~$ Start_HADOOP``

![starthadoop](/images/starthadoop.png)

###### Проверьте, запущены ли процессы Hadoop в master-namenode

``hdpuser@master-namenode:~$ jps``

![namenodejps](/images/namenodejps.png)

###### Проверьте, запущены ли процессы Hadoop на подчиненном сервере-datanode-1

``hdpuser@master-datanode-1:~$ jps``

![datanode1jps](/images/datanode1jps.png)

###### Проверьте, запущены ли процессы Hadoop на slave-datanode-2

``hdpuser@master-datanode-2:~$ jps``

![datanode2jps](/images/datanode2jps.png)

###### Веб-интерфейсы по умолчанию
> NameNode: http://master-namenode:9870/

![NameNode](/images/master-node9870.png)

> ResourceManager: http://master-namenode:8080/

![ResourceManager](/images/master-node8080.png)

###### Получить отчет

    hdpuser@master-namenode:~$ hdfs dfsadmin -report     --эта команда должна возвращать что-то вроде
    Configured Capacity: 59836907520 (55.73 GB)
    Present Capacity: 27630944256 (25.73 GB)
    DFS Remaining: 27630858240 (25.73 GB)
    DFS Used: 86016 (84 KB)
    DFS Used%: 0.00%
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

    Name: 192.168.1.72:9866 (master-namenode)
    Hostname: master-namenode
    Decommission Status : Normal
    Configured Capacity: 19945635840 (18.58 GB)
    DFS Used: 28672 (28 KB)
    Non DFS Used: 9707601920 (9.04 GB)
    DFS Remaining: 9201225728 (8.57 GB)
    DFS Used%: 0.00%
    DFS Remaining%: 46.13%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 1
    Last contact: Wed Apr 15 16:44:05 CEST 2020
    Last Block Report: Wed Apr 15 16:42:00 CEST 2020
    Num of Blocks: 0


    Name: 192.168.1.73:9866 (slave-datanode-1)
    Hostname: slave-datanode-1
    Decommission Status : Normal
    Configured Capacity: 19945635840 (18.58 GB)
    DFS Used: 28672 (28 KB)
    Non DFS Used: 9695444992 (9.03 GB)
    DFS Remaining: 9213382656 (8.58 GB)
    DFS Used%: 0.00%
    DFS Remaining%: 46.19%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 1
    Last contact: Wed Apr 15 16:44:04 CEST 2020
    Last Block Report: Wed Apr 15 16:41:56 CEST 2020
    Num of Blocks: 0


    Name: 192.168.1.74:9866 (slave-datanode-2)
    Hostname: slave-datanode-2
    Decommission Status : Normal
    Configured Capacity: 19945635840 (18.58 GB)
    DFS Used: 28672 (28 KB)
    Non DFS Used: 9692577792 (9.03 GB)
    DFS Remaining: 9216249856 (8.58 GB)
    DFS Used%: 0.00%
    DFS Remaining%: 46.21%
    Configured Cache Capacity: 0 (0 B)
    Cache Used: 0 (0 B)
    Cache Remaining: 0 (0 B)
    Cache Used%: 100.00%
    Cache Remaining%: 0.00%
    Xceivers: 1
    Last contact: Wed Apr 15 16:44:04 CEST 2020
    Last Block Report: Wed Apr 15 16:41:56 CEST 2020
    Num of Blocks: 0


###### Остановка Hadoop

``hdpuser@master-namenode:~$ Stop_HADOOP``

![stophadoop](/images/stophadoop.png)

&nbsp;

| :point_up:    | В следующем руководстве объясняется [как установить сервисы Spark Standalone и Hadoop Yarn в многоузловом кластере][nexttuto]. |
|---------------|:------------------------|

[nexttuto]: https://github.com/mnassrib/installing-spark-standalone-and-hadoop-yarn-on-cluster
