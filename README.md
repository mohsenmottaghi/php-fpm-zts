# Dockerfile PHP-FPM with ZTS
This repo cloned from `docker-library/php`

## Build Status
| Dockerfile | Base Image | PHP Version | Tag | Docker Hub Build | GitHub Build |
|--|--|--|--|--|--|
| 7.4/buster/Dockerfile | debian:buster-slim | 7.4 | latest | . | . |
| 7.4/buster/Dockerfile | debian:buster-slim | 7.4 | 7.4 | . | . |
| 7.3/buster/Dockerfile | debian:buster-slim | 7.3 | 7.3 | . | . |
| 7.2/buster/Dockerfile | debian:buster-slim | 7.2 | 7.2 | . | . |

## How to use this image
Create a Dockerfile in your PHP project.
```
FROM mohsenmottaghi/php-fpm-zts:latest
COPY . /var/www/html
```
Then, run the commands to build and run the Docker image:
```
$ docker build -t my-php-app .
$ docker run -it --rm --name my-running-app my-php-app
```
## How to install more PHP extensions
Many extensions are already compiled into the image, so it's worth checking the output of php -m or php -i before going through the effort of compiling more.

We provide the helper scripts docker-php-ext-configure, docker-php-ext-install, and docker-php-ext-enable to more easily install PHP extensions.

In order to keep the images smaller, PHP's source is kept in a compressed tar file. To facilitate linking of PHP's source with any extension, we also provide the helper script docker-php-source to easily extract the tar or delete the extracted source. Note: if you do use docker-php-source to extract the source, be sure to delete it in the same layer of the docker image.

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN docker-php-source extract \
    # do important things \
    && docker-php-source delete
```

### PHP Core Extensions
For example, if you want to have a PHP-FPM image with the gd extension, you can inherit the base image that you like, and write your own Dockerfile like this:

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd
```

Remember, you must install dependencies for your extensions manually. If an extension needs custom configure arguments, you can use the docker-php-ext-configure script like this example. There is no need to run docker-php-source manually in this case, since that is handled by the configure and install scripts.

If you are having difficulty figuring out which Debian or Alpine packages need to be installed before docker-php-ext-install, then have a look at [the install-php-extensions project](https://github.com/mlocati/docker-php-extension-installer). This script builds upon the docker-php-ext-* scripts and simplifies the installation of PHP extensions by automatically adding and removing Debian (apt) and Alpine (apk) packages. For example, to install the GD extension you simply have to run install-php-extensions gd. This tool is contributed by community members and is not included in the images, please refer to their Git repository for installation, usage, and issues.

See also "[Dockerizing Compiled Software](https://tianon.xyz/post/2017/12/26/dockerize-compiled-software.html)" for a description of the technique Tianon uses for determining the necessary build-time dependencies for any bit of software (which applies directly to compiling PHP extensions).

### Default extensions
Some extensions are compiled by default. This depends on the PHP version you are using. Run php -m in the container to get a list for your specific version.

### PECL extensions
Some extensions are not provided with the PHP source, but are instead available through PECL. To install a PECL extension, use pecl install to download and compile it, then use docker-php-ext-enable to enable it:

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN pecl install redis-5.1.1 \
    && pecl install xdebug-2.8.1 \
    && docker-php-ext-enable redis xdebug
```

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN apt-get update && apt-get install -y libmemcached-dev zlib1g-dev \
    && pecl install memcached-2.2.0 \
    && docker-php-ext-enable memcached
```

It is strongly recommended that users use an explicit version number in their pecl install invocations to ensure proper PHP version compatibility (PECL does not check the PHP version compatiblity when choosing a version of the extension to install, but does when trying to install it).

For example, memcached-2.2.0 has no PHP version constraints (https://pecl.php.net/package/memcached/2.2.0), but memcached-3.1.4 requires PHP 7.0.0 or newer (https://pecl.php.net/package/memcached/3.1.4). When doing pecl install memcached (no specific version) on PHP 5.6, PECL will try to install the latest release and fail.

Beyond the compatibility issue, it's also a good practice to ensure you know when your dependencies receive updates and can control those updates directly.

Unlike PHP core extensions, PECL extensions should be installed in series to fail properly if something went wrong. Otherwise errors are just skipped by PECL. For example, pecl install memcached-3.1.4 && pecl install redis-5.1.1 instead of pecl install memcached-3.1.4 redis-5.1.1. However, docker-php-ext-enable memcached redis is fine to be all in one command.

### Other extensions
Some extensions are not provided via either Core or PECL; these can be installed too, although the process is less automated:

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN curl -fsSL 'https://xcache.lighttpd.net/pub/Releases/3.2.0/xcache-3.2.0.tar.gz' -o xcache.tar.gz \
    && mkdir -p xcache \
    && tar -xf xcache.tar.gz -C xcache --strip-components=1 \
    && rm xcache.tar.gz \
    && ( \
        cd xcache \
        && phpize \
        && ./configure --enable-xcache \
        && make -j "$(nproc)" \
        && make install \
    ) \
    && rm -r xcache \
    && docker-php-ext-enable xcache
```

The `docker-php-ext-*` scripts can accept an arbitrary path, but it must be absolute (to disambiguate from built-in extension names), so the above example could also be written as the following:

```
FROM mohsenmottaghi/php-fpm-zts:latest
RUN curl -fsSL 'https://xcache.lighttpd.net/pub/Releases/3.2.0/xcache-3.2.0.tar.gz' -o xcache.tar.gz \
    && mkdir -p /tmp/xcache \
    && tar -xf xcache.tar.gz -C /tmp/xcache --strip-components=1 \
    && rm xcache.tar.gz \
    && docker-php-ext-configure /tmp/xcache --enable-xcache \
    && docker-php-ext-install /tmp/xcache \
    && rm -r /tmp/xcache
```
