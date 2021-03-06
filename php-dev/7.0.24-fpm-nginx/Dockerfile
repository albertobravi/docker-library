FROM php:7.0.24-fpm
MAINTAINER Daniele Piaggesi <daniele.piaggesi@bmeme.com>
ENV REFRESHED_AT 2017-11-03

# System env variables.
ENV DEBIAN_FRONTEND noninteractive
ENV XDEBUG_VERSION 2.5.5
ENV TIMEZONE "Europe/Rome"
ENV MAILHOG_VERSION "v0.1.9"

# PHP env variables.
ENV COMPOSER_HOME "/.composer"
ENV PHP_MEMORY_LIMIT "512M"
ENV PHP_ERROR_LOG "/tmp/php_errors.log"
ENV UPLOAD_MAX_FILESIZE "20M"
ENV POST_MAX_SIZE "20M"
ENV AUTO_PREPEND_FILE "${COMPOSER_HOME}/vendor/autoload.php"

# Download basic packages
RUN apt-get update && apt-get install -y build-essential wget libpcre3-dev

# Download and add dotdeb packages
RUN echo "deb http://packages.dotdeb.org jessie all" >> /etc/apt/sources.list.d/dotdeb.list && \
	echo "deb-src http://packages.dotdeb.org jessie all" >> /etc/apt/sources.list.d/dotdeb.list && \
	wget -O- http://www.dotdeb.org/dotdeb.gpg | apt-key add -

# Download and add ansible PPA packages
RUN echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list.d/ansible.list && \
      apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367

# Build stack
RUN apt-get update \
    && apt-get install -y libpng12-0 \
        libpng12-dev \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng12-dev \
        git \
        vim \
        zip \
        unzip \
        mysql-client \
        cron \
        rsyslog \
        supervisor \
        libpq-dev \
        zlib1g-dev \
        libicu-dev \
        nginx \
        g++ \
        graphviz \
        ansible \
        && dpkg-reconfigure -f noninteractive tzdata \
        && apt-get autoremove -y \
        && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install php libraries and configure PHP
RUN docker-php-ext-install pdo_mysql \
    && docker-php-ext-install -j$(nproc) json iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install zip \
    && docker-php-ext-install bcmath \
    && docker-php-ext-install mbstring \
    && docker-php-ext-install mcrypt \
    && docker-php-ext-install sockets \
    && docker-php-ext-install intl \
    && pecl install xdebug-${XDEBUG_VERSION} \
    && pecl install oauth \
    && echo "${TIMEZONE}" > /etc/timezone \

# Install composer and other stuff
RUN mkdir ${COMPOSER_HOME} \
    && curl -k -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && composer global require symfony/var-dumper \
    && chown -R www-data:www-data ${COMPOSER_HOME} \
    && curl -fSL "https://github.com/mailhog/MailHog/releases/download/${MAILHOG_VERSION}/MailHog_linux_amd64" -o /usr/local/bin/mailhog \
    && chmod +x /usr/local/bin/mailhog \
    && ansible-galaxy install carlosbuenosvinos.ansistrano-deploy carlosbuenosvinos.ansistrano-rollback

# Create SSH structure
RUN mkdir /root/.ssh && chmod 700 /root/.ssh

# Configure php.
COPY config/php/*.ini /usr/local/etc/php/conf.d/
RUN echo "clear_env = no" >> /usr/local/etc/php-fpm.d/www.conf

# Configure Nginx
COPY config/nginx/default.conf /etc/nginx/sites-available/default

# Configure supervisord
RUN mkdir -p /var/log/supervisor
ADD config/supervisor/supervisord.conf /etc/supervisord.conf

# Configure cron.
COPY config/cron/crontab /etc/cron.d/cron
RUN rm -Rf /etc/cron.daily  && \
    rm -Rf /etc/cron.weekly && \
    rm -Rf /etc/cron.monthly && \
    rm -Rf /etc/cron.hourly && \
    chmod a+x /etc/cron.d/cron

# Install blackfire agent.
RUN VERSION=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION;") \
    && curl -A "Docker" -o /tmp/blackfire-probe.tar.gz -D - -L -s https://blackfire.io/api/v1/releases/probe/php/linux/amd64/${VERSION} \
    && tar zxpf /tmp/blackfire-probe.tar.gz -C /tmp \
    && mv /tmp/blackfire-*.so $(php -r "echo ini_get('extension_dir');")/blackfire.so \
    && printf "extension=blackfire.so\nblackfire.agent_socket=tcp://blackfire:8707\n" > ${PHP_INI_DIR}/conf.d/blackfire.ini

# Install ngrok.
ADD https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip /ngrok.zip
RUN unzip /ngrok.zip -d /bin \
    && rm -f ngrok.zip \
    && touch /.ngrok

# Add PHPINFO
COPY scripts/index.php /var/www/html/index.php

COPY scripts/start.sh /start.sh
RUN chmod +x /start.sh

EXPOSE 80 443 9001

CMD ["/start.sh"]
