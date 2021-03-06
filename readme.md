
# Table of Contents

1.  [Задание](#orgd2a68f1)
2.  [Решение](#orgca0aa39)


<a id="orgd2a68f1"></a>

# Задание

Это docker-образ с приложением, которое слегка поломано. Починить его — и есть тестовое задание для ops-инженера =)

Запустить его можно так:

    docker run -p 3000:80 -it ecwid/ops-test-task:20210311a

(в версии "20210311" был баг в интерфейсе)

После чего приложение доступно на <http://localhost:3000>. Ну не сразу доступно, придётся повозиться.

Когда тебе удастся это сделать, присылай на join@ecwid.com:

-   код подтверждения, который получишь в рабочем приложении,
-   краткий и понятный отчёт-прохождение (представь, что пишешь коммент(ы) в баг-трекере),
-   новый Dockerfile на основе данного docker-образа с починкой всех проблем.

Не всё получилось? Не страшно, присылай, что есть.

Знаешь, как улучшить задание? Пиши email, просто поболтаем.

Смотри [наши вакансии на hh.ru](https://ulyanovsk.hh.ru/search/vacancy?st=searchVacancy&text=DevOps+%D0%AD%D0%BA%D0%B2%D0%B8%D0%B4&salary=&currency_code=RUR&experience=doesNotMatter&order_by=relevance&search_period=0&items_on_page=50&no_magic=true&L_save_area=true), там вкратце о том, кого мы ищем и что предстоит делать ;)


<a id="orgca0aa39"></a>

# Решение

Запускаю контейнер:

    docker run -p 3000:80 -it --rm --name ecwid ecwid/ops-test-task:20210311a

На странице <http://localhost:3000> вижу ошибку 502. Запускаю в этом контейнере bash:

    docker exec -it ecwid /bin/bash

Смотрю конфиг nginx. Вижу, что он отправляет запросы на <http://localhost:8000>. Смотрю открытые порты:

    root@5b0d0e063df0:/# netstat -ntap
    Active Internet connections (servers and established)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      43/nginx: master pr
    tcp        0      0 127.0.0.1:8082          0.0.0.0:*               LISTEN      22/java
    tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      -
    tcp6       0      0 :::80                   :::*                    LISTEN      43/nginx: master pr

На порту 8000 нет ничего, но какой-то сервис висит на 8082. Меняю порт назначения в конфиге nginx и перезапускаю его. Снова иду на <http://localhost:3000> и вижу пустую страницу. Смотрю логи box. Вижу, что ему не хватило памяти. Увеличиваю ему память до 250 МБ в /etc/init.d/box и перезапускаю его.
Обновляю <http://localhost:3000>. Вижу чёрный квадрат с 3 пройденными и 2 заваленными проверками. Смотрю логи постгреса. Последовательно добавляю: пользователя (пароль из файла /etc/box.properties), базу данных, таблицу (схема таблицы /opt/box/schema.sql), своё мыло в неё. Получаю код: 54502917495705063000237613323081902911576395265967119224924316489932875339594
Первый вариант решения:

    FROM ecwid/ops-test-task:20210311a
    
    RUN \
    sed -i "s/8000/8082/" /etc/nginx/sites-available/box.conf && \
    sed -i "s/JAVA_OPTS=.*/JAVA_OPTS='-Xmx250m'/" /etc/init.d/box && \
    service postgresql start && \
    echo "CREATE DATABASE box" | su - postgres -c psql && \
    su - postgres -c 'psql -d box' < /opt/box/schema.sql && \
    echo "INSERT INTO devops VALUES('gavk@ngs.ru');" | su - postgres -c 'psql -d box' && \
    echo "CREATE USER box WITH PASSWORD 'iwwIEIeEiEDDecIEeIwC'; GRANT SELECT on devops to box" | su - postgres -c 'psql -d box' && \
    service postgresql stop
    
    ENTRYPOINT ["/startup"]

Второй вариант решения:

    FROM ecwid/ops-test-task:20210311a AS builder
    
    RUN \
    sed -i "s/8000/8082/" /etc/nginx/sites-available/box.conf && \
    sed -i "s/JAVA_OPTS=.*/JAVA_OPTS='-Xmx250m'/" /etc/init.d/box && \
    service postgresql start && \
    echo "CREATE DATABASE box" | su - postgres -c psql && \
    su - postgres -c 'psql -d box' < /opt/box/schema.sql && \
    echo "INSERT INTO devops VALUES('gavk@ngs.ru');" | su - postgres -c 'psql -d box' && \
    echo "CREATE USER box WITH PASSWORD 'iwwIEIeEiEDDecIEeIwC'; GRANT SELECT on devops to box" | su - postgres -c 'psql -d box' && \
    service postgresql stop
    
    FROM ecwid/ops-test-task:20210311a
    COPY --from=builder /etc/nginx/sites-available/box.conf /etc/nginx/sites-available/
    COPY --from=builder /etc/init.d/box /etc/init.d
    COPY --from=builder /var/lib/postgresql/12 /var/lib/postgresql/12
    
    ENTRYPOINT ["/startup"]

