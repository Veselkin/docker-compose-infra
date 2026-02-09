FROM php:8.2-fpm

# Install dependencies for PHP extensions
RUN apt-get update && apt-get install -y \
    libonig-dev \
    libicu-dev \
    libzip-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libwebp-dev \
    libfreetype6-dev \
    libxml2-dev \
    libcurl4-openssl-dev \
    unzip \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Configure and install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) \
        pdo \
        pdo_mysql \
        mbstring \
        intl \
        zip \
        gd \
        curl \
        dom \
        xml \
        xmlwriter \
        simplexml \
    && docker-php-ext-enable \
        mbstring \
        intl \
        curl \
        dom \
        xml \
        xmlwriter \
        simplexml

# Check for iconv (typically built-in)
RUN php -r "if (!extension_loaded('iconv')) { exit(1); }" || echo "Iconv not available"

# Create and set permissions for sessions directory
RUN mkdir -p /tmp/sessions && chown -R www-data:www-data /tmp/sessions && chmod -R 700 /tmp/sessions

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Verify installed extensions and Composer
RUN php -m | grep -E "(curl|dom|gd|simplexml|xml|zip|xmlwriter|mbstring|intl|pdo)" \
    && composer --version

# Set www-data user and group to UID/GID 1000 for better host compatibility
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data