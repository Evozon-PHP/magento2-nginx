# Magento2 - Nginx configuration

Optimised Nginx configuration for Magento2

### Reason

Magento's sample configuration file for Nginx is just for local testing, but not production ready. It lacks many performance and security directives.

### Goals

* Protect as many resources as possible.
* Use HTTP caching for as many resources as possible.
* Keep configuration simple. (`/media/` and `/static/` really make it hard in Magento2).
* Correct use of `IF` statements. Otherwise it's [evil](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/).

We will update and adapt the current configuration as we go. Feel free to use, test, ask, report bugs and, if possible, contribute.

### Requirements

* Nginx `1.9.5` or higher.
* Magento2 (tested currently on `2.1.1 CE`)
* PHP-FPM (might work as reverse proxy, but you'll have to change some files and directives).

### Installation

* Install Nginx and overwrite it's files with the ones provided here.
* Configure your PHP-FPM upstream (`conf.d/upstream-magento2.conf`).
* Read the notes below and modify where needed.

### To do

* Limit the request processing rate for bots and spiders on expensive operations (search, layered navigation, etc.).
* Find a smarter approach for maintenance status check and bypassing.
* Find a smarter approach for static resources in `/media`, `/static`, `/setup`, `/update` and everything outside.

### Notes

##### Magento root folder

Edit `sites-enabled/magento2.conf` and set `$MAGE_ROOT` to the root folder of your Magento2 installation. Do not include the `/pub` suffix as it will be added automatically.

##### Magento mode

Edit `sites-enabled/magento2.conf` and set `$MAGE_MODE`. Available options are: `default`, `development` and `production`. You might leave it as `default` and set mode via `bin/magento deploy:mode:set`.

##### Multi-store

To enable multi-store support, edit `conf.magento2.d/multistore.conf` and configure domains to populate `$MAGE_RUN_CODE` and `$MAGE_RUN_TYPE` according to your needs. There's no need to edit any PHP files and it looks way better.

##### Magento setup and upgrade

While there are two routes, with limited access, `/setup/index.php` and `/update/index.php`, we strongly encourage you to use the CLI tool instead.

```
bin/magento setup:install
bin/magento setup:upgrade
```

##### Admin panel restriction

We strongly encourage you to use a secret route to your admin panel. Magento2 usually generates one in form of `/admin_xxxxx`, but you are free to change that via the `app/etc/env.php` file (`backend > frontName` node). Run `bin/magento info:adminuri` to get your admin route.

By default, we have added IP address restriction to the admin panel. Only a predefined list of IP addresses are allowed to access it, everyone else is blocked.

Edit `location.magento2.d/admin.conf` and set your IP address via the [allow](http://nginx.org/en/docs/http/ngx_http_access_module.html#allow) directive. In case you have a custom route, this is where you have to configure it (hint: [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)).

##### Static error pages

Error pages are usually handled by Magento and PHP (e.g. `http://domain.com/errors/403.php`). We strongly disagree with this, because it cannot handle PHP errors and it is an expensive operation.

Edit `sites-enabled/magento2.conf` and set `$MAGE_ROOT_ERRORS` to a folder where you create pure static error pages. Add here at least `403.html`, `404.html` and `503.html`. Style them as desired, but try to keep them as simple as possible.

Edit `location.magento2.d/error.conf` and go through each named error location and remove the [try_files](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files) directive to PHP files, then uncomment [root](http://nginx.org/en/docs/http/ngx_http_core_module.html#root) and [try_files](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files) directive forwarding to the static error page.

```
# Magento2 error page (worst performance)
#try_files $uri /errors/503.php;

# Static error page (best performance)
root $MAGE_ROOT_ERRORS;
try_files /503.html @error;
```
##### Nginx and PHP-FPM status pages

Edit `location.magento2.d/status.conf` to configure access to the Nginx's status page. Make sure [ngx_http_stub_status_module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) is enabled.

For PHP-FPM status and ping pages, you have to enable them first in your PHP-FPM pool configuration. See `pm.status_path = /status` and `ping.path = /ping`.

All status routes have limited [access](http://nginx.org/en/docs/http/ngx_http_access_module.html). Add your IP address to the whitelist.

##### Security

* Make sure to always restrict access as strict as possible: status pages, admin panel, update and any other resource that is not important for execution.
* Restrict resources to be requested from other domains by enabling [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) (`add_header Access-Control-Allow-Origin "www.domain.com";`).
* Try to make use of the `Content-Security-Policy` HTTP header. It's tough, but you may find help at [cspisawesome.com](http://cspisawesome.com/).

##### Performance

* Set `Expires` and `Cache-Control` headers correctly and allow clients to cache the resources.
* Merge and minify your CSS & JS.
* Compile DI (bin/magento setup:di:compile).
* Use PHP 7+ with [PHP-FPM](https://php-fpm.org/) and [OPcache](http://php.net/opcache).
* Always deploy your static content (`bin/magento setup:static-content:deploy`). Otherwise each asset will be forwarded on it's first request to a PHP entry point (`/get.php` & `/static.php`) to be generated on disk. Takes time and precious resources.
* Enable `sendfile`, `tcp_nopush`, `tcp_nodelay`.
* Use the best [connection processing method](http://nginx.org/en/docs/events.html) for your OS.
* Experiment with [asynchronous file I/O](http://nginx.org/en/docs/http/ngx_http_core_module.html#aio).

### Special Thanks

* Magento2 developers and contributers who worked on the [original](https://github.com/magento/magento2/blob/develop/nginx.conf.sample) configuration.
* All contributors to [HTML5 Boilerplate Nginx server configs](https://github.com/h5bp/server-configs-nginx/).
* [MagenX](https://github.com/magenx/Magento-nginx-config/) for one of the configurations we used for inspiration.
* [K&T Host](https://www.knthost.com/nginx/blocking-bots-nginx) for the comprehensive bots and spiders list.
* Everyone else on the internet, who posted and answer, article, post or piece of code that helped us out.
