CacheControlBundle
==================

This Bundle provides a way to set path based cache expiration headers via the app configuration

Installation
============

    1. Add this bundle to your project as a Git submodule:

        $ git submodule add git://github.com/liip/CacheControlBundle.git vendor/bundles/Liip/CacheControlBundle

    2. Add the Liip namespace to your autoloader:

        // app/autoload.php
        $loader->registerNamespaces(array(
            'Liip' => __DIR__.'/../vendor/bundles',
            // your other namespaces
        ));

    3. Add this bundle to your application's kernel:

        // application/ApplicationKernel.php
        public function registerBundles()
        {
          return array(
              // ...
              new Liip\CacheControlBundle\LiipCacheControlBundle(),
              // ...
          );
        }

Cache control
=============

Simply configure as many paths as needed with the given cache control rules and/or the location
of the varnish reverse proxies:

    # app/config.yml
    liip_cache_control:
        rules:
            # the controls section values are used in a call to Response::setCache();
            - { path: /, controls: { public: true, max_age: 15, s_maxage: 30, last_modified: "-1 hour" } }
        varnish:
            domain: http://www.liip.ch
            ips: 10.0.0.10, 10.0.0.11 # comma separated list of ips, or an array of ips
            port: 80  # port varnish is listening on for incoming web connections
        authorization_listener: true

Varnish helper
==============

Purging
-------

Please add the following code to your Varnish configuration.

    #top level:
    # who is allowed to purge from cache
    # http://varnish-cache.org/trac/wiki/VCLExamplePurging
    acl purge {
        "127.0.0.1"; #localhost for dev purposes
        "10.0.11.0"/24; #server closed network
    }

    #in sub vcl_recv
    # purge if client is in correct ip range
    if (req.request == "PURGE") {
        if (!client.ip ~ purge) {
            error 405 "Not allowed.";
        }
        purge("req.url ~ " req.url);
        #log "PURGE " req.url;
        error 200 "Success";
    }

NOTE: this code invalidates the url for all domains. If your varnish serves multiple domains,
you should improve this configuration. Pull requests welcome :-)

The varnish path invalidation is about equivalent to doing this:

     netcat localhost 6081 << EOF
     PURGE /url/to/purge HTTP/1.1
     Host: webapp-host.name

     EOF

To use the varnish cache helper you must inject the ``liip_cache_control.varnish`` service
or fetch it from the service container:

    // using a "manual" url
    $varnish = $this->container->get('liip_cache_control.varnish');
    $varnish->invalidatePath('/some/path');

    // using the router to generate the url
    $router = $this->container->get('router');
    $varnish = $this->container->get('liip_cache_control.varnish');
    $varnish->invalidatePath($router->generate('myRouteName'));

Force refresh
-------------

Alternatively one can also force a refresh using the approach

    #top level:
    # who is allowed to purge from cache
    # http://www.varnish-cache.org/trac/wiki/VCLExampleEnableForceRefresh
    acl refresh {
        "127.0.0.1"; #localhost for dev purposes
        "10.0.11.0"/24; #server closed network
    }

    sub vcl_hit {
        if (!obj.cacheable) {
            pass;
        }

        if (req.http.Cache-Control ~ "no-cache" && client.ip ~ refresh) {
            set obj.ttl = 0s;
            return (restart);
        }
        deliver;
    }

The vanish path force refresh is about equivalent to doing this:

    netcat localhost 6081 << EOF
    GET /url/to/refresh HTTP/1.1
    Host: webapp-host.name
    Cache-Control: no-cache, no-store, max-age=0, must-revalidate

    EOF

To use the varnish cache helper you must inject the ``liip_cache_control.varnish`` service
or fetch it from the service container:

    // using a "manual" url
    $varnish = $this->container->get('liip_cache_control.varnish');
    $varnish->refreshPath('/some/path');

Cache authorization listener
============================

This listener makes it possible to stop a request with a 200 "OK" for HEAD requests
right after the security firewall has finished. This is useful when one uses Varnish while
handling content that is not available for all users.

In this scenario on a cache hit, Varnish can be configured to issue a HEAD request when this
content is accessed. This way Symfony2 can be used to validate the authorization, but no
work needs to be made to regenerate the content that is already in the Varnish cache.

Note this obviously means that it only works with path based Security. Any additional security
implemented inside the Controller will be ignored.

Note further that a HEAD response is supposed to contain the same HTTP header meta data as the
GET response to the same URL. However for the purpose of this use case we have no other choice
but to assume a 200.

TODO: add example Varnish config
