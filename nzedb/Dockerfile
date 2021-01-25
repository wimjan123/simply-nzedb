FROM ubuntu:18.04
MAINTAINER thewillonline

# Configure Timezone
ENV TZ 'Europe/Amsterdam'
RUN echo $TZ > /etc/timezone && \
apt-get update && apt-get install -y tzdata && \
rm /etc/localtime && \
ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && \
dpkg-reconfigure -f noninteractive tzdata && \
apt-get clean

RUN apt-get update -y && \
    apt-get install -y systemd software-properties-common && \
    add-apt-repository -y ppa:ondrej/php && \
    apt-add-repository multiverse && \
    # apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc' && \
    # add-apt-repository 'deb [arch=amd64,arm64,ppc64el] https://mirror.zol.co.zw/mariadb/repo/10.5/ubuntu bionic main' && \
    apt-get update -y && \
    apt-get upgrade -y && \
    apt autoremove -y

#installing apache2
RUN apt-get install -y \
  apache2 apache2-utils

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
COPY . /var/www/html/
EXPOSE 80

RUN service apache2 start && \
  chown www-data /var/www/html/ -R

#Instaling MariaDB
RUN apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8 && \
  add-apt-repository -y 'deb [arch=amd64] http://mirror.zol.co.zw/mariadb/repo/10.3/ubuntu bionic main' && \
  apt update -y && \
  apt install -y mariadb-server mariadb-client && \
  service mysql start

#mysql_secure_installation
ADD https://github.com/wimjan123/simply-nzedb/blob/master/nzedb/automate_mysqlsecure.sh /tmp/
RUN chmod +x /tmp/automate_mysqlsecure.sh && ./tmp/automate_mysqlsecure.sh

#s6 installer
ADD https://github.com/just-containers/s6-overlay/releases/download/v2.2.0.1/s6-overlay-amd64-installer /tmp/
RUN chmod +x /tmp/s6-overlay-amd64-installer && ./tmp/s6-overlay-amd64-installer /

# vnstat in testing repo

# mytop + deps
RUN apt-get -y install \
  mariadb-server  \
  perl \
  libmysqlclient-dev \
  libterm-readkey-perl

# Install composer
RUN curl https://getcomposer.org/installer | php7.3 -- --install-dir=/usr/bin --filename=composer --version=1.10.19

# Install Python MySQL Modules
RUN export LC_ALL="en_US.UTF-8"
RUN pip3 install --upgrade pip && \
  pip3 install --upgrade setuptools && \
  pip3 install cymysql pynntp

# Configure PHP
RUN sed -ri 's/(max_execution_time =) ([0-9]+)/\1 120/' /etc/php/7.3/fpm/php.ini && \
  sed -ri "s/(memory_limit =) (.*$)/\1 -1/" /etc/php/7.3/fpm/php.ini && \
  sed -ri 's/;(date.timezone =)/\1 Europe\/Amsterdam/' /etc/php/7.3/fpm/php.ini && \
  sed -ri 's/listen\s*=\s*127.0.0.1:9000/listen = 9000/g' /etc/php/7.3/fpm/pool.d/www.conf && \
  sed -ri 's|;include_path = ".:/php/includes"|include_path = ".:/usr/share/php7.3"|g' /etc/php/7.3/fpm/php.ini && \
  mkdir -p /var/log/php7.3-fpm/ && \
  ln -sf /dev/stdout /var/log/php7.3-fpm.log && \
  ln -s /usr/sbin/php-fpm7.3 /usr/sbin/php-fpm7

# Install and configure nginx.
RUN mkdir -p /var/log/nginx && \
    mkdir -p /etc/nginx && \
    mkdir -p /tmp/nginx && \
    chmod 755 /var/log/nginx && \
    chmod 777 /tmp && \
    touch /var/log/nginx/nginx-error.log && \
    ln -sf /dev/stderr /var/log/nginx/nginx-error.log &&\
    ln -sf /dev/stdout /var/log/nginx/nginx-access.log

# Clone nZEDb and set directory permissions
ENV NZEDB_VERSION "0.x"
RUN mkdir -p /var/www && \
  cd /var/www && \
  git clone https://github.com/nZEDb/nZEDb.git && \
  cd /var/www/nZEDb && \
  git checkout --quiet --force $NZEDB_VERSION && \
  composer install --prefer-dist && \
  chmod -R 777 /var/www/nZEDb/ && \
  # nuke all git repos' .git dir except for nzedb's .git dir to save space
  find . -name ".git" -type d | grep -v "\.\/\.git" | xargs rm -rf && \
  # nuke ~350MB of composer cache
  composer clear-cache

# install tmux
RUN apt install -y tmux


# install par2
RUN apt-get install -y par2

# Create dir for importing nzbs
RUN mkdir -p /var/www/nZEDb/resources/import

# Switch out php executable to instrument invocations
RUN mv /usr/bin/php /usr/bin/php.real
COPY php.proxy /usr/bin/php

# Use pigz (parallel gzip) instead of gzip to speed up db backups
RUN mv /bin/gzip /bin/gzip.real && \
  ln -s /usr/bin/pigz /bin/gzip

# iconv has issues in musl which affects NFO conversion to include
# cool ascii chars. Remove the problematic parts - TRANSLIT and IGNORE
# See https://github.com/slydetector/simply-nzedb/issues/31
RUN sed -i "s|UTF-8//IGNORE//TRANSLIT|UTF-8|g" /var/www/nZEDb/nzedb/utility/Text.php

LABEL nzedb=$NZEDB_VERSION \
  maintainer=thewillonline \
  url=https://github.com/wimjan123/simply-nzedb

RUN mkdir -p /var/www/nZEDb/resources/tmp && chmod 777 /var/www/nZEDb/resources/tmp

ENV TERM tmux
EXPOSE 8800
ADD s6 /etc/s6
CMD ["/bin/s6-svscan","/etc/s6"]
WORKDIR /var/www/nZEDb/misc/update
