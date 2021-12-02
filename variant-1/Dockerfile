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
