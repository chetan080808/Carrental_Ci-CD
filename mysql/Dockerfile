FROM mysql:8.0

# Copy the SQL dump so MySQL auto-imports it on first run
COPY carrental.sql /docker-entrypoint-initdb.d/carrental.sql

EXPOSE 3306
