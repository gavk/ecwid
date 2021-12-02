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
