# containers04
Лабораторная работа №4: Запуск сайта в контейнере

Цель работы: Настройка и запуск веб-сайта на базе Apache HTTP Server + PHP (mod_php) + MariaDB в Docker containere.

Скачиваем и устанавливаем Docker.
Регистрируемся в Docker.
Запускаем Docker.
Создаем Dockerfile с настройками.

# create from debian image
FROM debian:latest

# install apache2, php, mod_php for apache2, php-mysql and mariadb
RUN apt-get update && \
    apt-get install -y apache2 php libapache2-mod-php php-mysql mariadb-server supervisor

Выполнить команду в папке с докер файлом

docker build -t containers04-01 .

После выполнения команд у нас появятся новые файлы в папках.

Конфигурационные файлы.

Hачинаем с Apache2.

Файл: files/apache2/000-default.conf

1) Настроим #ServerName на адрес, по которому мы будем открывать наш вебсайт (localhost).
2) ServerAdmin - укажем нашу электронную почту. Этот метод устарел и почти не используется, но в старых версиях он устанавливает $_SERVER['SERVER_ADMIN'].
3) DocumentRoot - установим корневой путь к файлам для работы вебсайта.
Установим параметр DirectoryIndex на index.php index.html.

files/apache2/apache2.conf
1) Настроим #ServerName на адрес, по которому мы будем открывать наш вебсайт (localhost).

Далее PHP.

files/php/php.ini
1) Настроим error_log на /var/log/php_errors.log.
2) Установим лимиты для параметров memory_limit, upload_max_filesize, post_max_size, max_execution_time как 128M, 128M, 128M, 120 секунд соответственно.

Далее MariaDB.

files/mariadb/50-server.cnf

1) Раскомментируем и настроим error_log как /var/log/mysql/error.log.

Создаем файл конфигураций для supervisord
supervisord.conf

[supervisord]
nodaemon=true
logfile=/dev/null
user=root

# apache2
[program:apache2]
command=/usr/sbin/apache2ctl -D FOREGROUND
autostart=true
autorestart=true
startretries=3
stderr_logfile=/proc/self/fd/2
user=root

# mariadb
[program:mariadb]
command=/usr/sbin/mariadbd --user=mysql
autostart=true
autorestart=true
startretries=3
stderr_logfile=/proc/self/fd/2
user=mysql

Обновляем Dockerfile и добавляем следующий код сразу после строки FROM

# mount volume for mysql data
VOLUME /var/lib/mysql

# mount volume for logs
VOLUME /var/log

И проверяем, что у нас установлен supervisord как инструкция.

Затем добавляем копирование веб-сайта WordPress.

# add wordpress files to /var/www/html
ADD https://wordpress.org/latest.tar.gz /var/www/html/

Добавляем инструкцию для копирования обновленных файлов.

# copy the configuration file for apache2 from files/ directory
COPY files/apache2/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY files/apache2/apache2.conf /etc/apache2/apache2.conf

# copy the configuration file for php from files/ directory
COPY files/php/php.ini /etc/php/8.2/apache2/php.ini

# copy the configuration file for mysql from files/ directory
COPY files/mariadb/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf

# copy the supervisor configuration file
COPY files/supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

Далее создаем папку 

# create mysql socket directory
RUN mkdir /var/run/mysqld && chown mysql:mysql /var/run/mysqld

Прописываем команду для запуска запуска supervisord
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

Выполняем код через программу Docker. Во вкладке "Exec" нужно выполнить следующие команды:
"mysql", а затем:
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'localhost' IDENTIFIED BY 'wordpress';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'localhost';
FLUSH PRIVILEGES;
EXIT;

При выполнении данной процедуры мы изменили следующие файлы.

000-default.conf
apache2.conf

php.ini
50-server.cnf

supervisord.conf
wp-config.php


Параметр DirectoryIndex отвечает за изначальный файл, который должен запускаться при открытии вебсайта.

Файл wp-config.php отвечает за настройки WordPress, такие как настройка базы данных или другие специальные данные.

Параметр post_max_size отвечает за максимальный размер загружаемых/обрабатываемых данных через метод POST.

Я считаю, что создание образа контейнера - это отличный метод для программирования и тестирования настройки веб-сервисов как в Amazon Web Services, так и в Google Cloud Instance. Единственный недостаток заключается в том, что могут возникнуть ошибки конфигурации сети или при работе с большими объемами данных.
