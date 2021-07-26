# Heroku WP

This is a template for installing and running [WordPress](http://wordpress.org/) on [Heroku](http://www.heroku.com/) with a focus on speed and security while using the official Heroku stack.

## About

The repository is built on top of the standard Heroku PHP buildpack so you don't need to trust some sketchy 3rd party s3 bucket.

- [NGINX](http://nginx.org) - Fast scalable webserver.
- [PHP 7](http://php.net) - Latest and greatest with performance on par with HHVM.
- [Composer](https://getcomposer.org) - A dependency manager to make installing and managing plugins easier.

Heroku WP uses the following addons:

- [MariaDB](https://mariadb.org) / [jawsdb-maria](https://elements.heroku.com/addons/jawsdb-maria) - A binary compatible MySQL replacement with even better performance.
- [Redis](http://redis.io) / [heroku-redis](https://elements.heroku.com/addons/heroku-redis) - An in-memory datastore for fast persistant object cache.

In addition repository comes bundled with the following tools and must use plugins.

- [WP-CLI](http://wp-cli.org) - For simple management of your WP install.
- [Redis Object Cache](http://wordpress.org/plugins/redis-cache) - For using Redis as a persistant, shared, object cache.
- [Secure DB Connection](http://wordpress.org/plugins/secure-db-connection) - For ensuring connections to the database are secure and encrypted.

Finally these plugins are pre-installed as they are highly recommended but not activated.

- [S3 Uploads](https://github.com/humanmade/S3-Uploads)

WordPress and most included plugins are installed by Composer on build. To add new plugins or upgrade versions of plugins simply update the `composer.json` file and then generate the `composer.lock` file with the following command locally:

    composer update --ignore-platform-reqs

To add local plugins and themes, you can create `plugins/` and `themes/` folders inside `/public/wp-content` which upon deploy to Heroku will be copied on top of the standard WordPress install, themes, and plugins specified by Composer.

### Media Uploads

[S3 Uploads](https://github.com/humanmade/S3-Uploads) is a lightweight "drop-in" for storing WordPress uploads on [Amazon S3](http://aws.amazon.com/s3) instead of the local filesystem. If you want media uploads you must activate this plugin and configure a S3 bucket because the local filesystem for Heroku Dynos are ephemeral.

To activate this plugin:

First set your S3 credentials via Heroku configs with AWS S3 path-style URLs. It's best practices to URL encode your AWS key and secret, (e.g. use `%2B` for `+` and `%2F` for `/`,) however non URL encoded values should still work even if they are invalid URLs.

    heroku config:set AWS_S3_URL="s3://{AWS_KEY}:{AWS_SECRET}@s3.amazonaws.com/{AWS_BUCKET}"

If you would like to set the optional region for your S3 bucket use the region-specific endpoint.

    heroku config:set AWS_S3_URL="s3://{AWS_KEY}:{AWS_SECRET}@s3-{AWS_REGION}.amazonaws.com/{AWS_BUCKET}"

For example, if your bucket is in the South America (São Paulo) region use:

    heroku config:set AWS_S3_URL="s3://my-key:my-secret@s3-sa-east-1.amazonaws.com/my-bucket"

If you would like to use a custom bucket URL either because you are proxying the requests or if you are using a domain for the bucket name you can do so by setting a custom url param:

    heroku config:set AWS_S3_URL="s3://{AWS_KEY}:{AWS_SECRET}@s3-{AWS_REGION}.amazonaws.com/{AWS_BUCKET}?url={BUCKET_URL}"

The `BUCKET_URL` should have a scheme attached, e.g.:

    heroku config:set AWS_S3_URL="s3://my-key:my-secret@s3-sa-east-1.amazonaws.com/my-bucket?url=https://static.example.com"

Then activate the plugin in WP Admin.

### Securing the database connection using an SSL certificate

This repo already comes with the Amazon RDS root CAs installed for securing DB connections.

To turn on SSL simply set the `WP_DB_SSL` config.

    heroku config:set WP_DB_SSL="ON"

If you use another MySQL database and have a self signed cert you can add the self signed CA to the trusted store by committing it to `/support/mysql-certs` and setting the filenames or explicitly setting it in the ENV config itself:

    heroku config:set MYSQL_SSL_CA="$(cat /path/to/server-ca.pem)"

### Offloaded WP Cron

WP Cron relies on visitors to the site to trigger scheduled jobs to run which can be a problem for lightly trafficked sites. Instead we can have an external jobs system (IronWorker) run WP Cron on schedule to provide consistency.

For this reason Heroku Scheduler is used with a frequency of **10 minutes** along with the following command:

    curl -fsSL https://example.com/wp-cron.php?doing_wp_cron

Where example.com is the domain you are using for your website.

## Usage

Because a file cannot be written to Heroku's file system, updating and installing plugins or themes should be done locally and then pushed to Heroku. Even better would be to use Composer to install plugins so that version control and upgrading is simply a matter of editing the `composer.json` file and bumping the version number.

## Internationalization

In most cases you may want to have your WordPress blog in a language different than its default (US English). In that case all you need to do is download the .mo and .po files for your language from [wpcentral.io/internationalization](http://wpcentral.io/internationalization/) and place them in the `languages` directory you'll create under `public/wp-content`. Then you should commit changes to your local branch and push them to your heroku remote. After that, you'll be able to select the new language from the WP admin panel.

## Updating WordPress

In order to update the WordPress installation you need to use composer.

**NOTE: It is highly recommended to make a backup of the database before proceeding.**

Edit the file and change the version of the `wordpress/wordpress` package and then change the zip path and version under:

    {
        "type": "package",
        "package": {
        "name": "wordpress/wordpress",
        "version": "5.8.0",
        "dist": {
            "type": "zip",
            "url": "https://github.com/WordPress/WordPress/archive/refs/tags/5.8.zip"
        }
    }

After pushing changes update the WordPress database via WP-CLI:

    $ heroku run wp core update --app <app>
    Success: WordPress is up to date.

WordPress will prompt for updating the database. After that you'll be good to go.

## Custom Domains

Heroku allows you to add custom domains to your site hosted with them. To add your custom domain, enter in the follow commands.

    $ heroku domains:add www.example.com
    > Added www.example.com as a custom domain name to myapp.heroku.com

### Clean the cache

One of the most common problems is if you make a DB change but still have stale cache refering to the old configs the easiest way to fix this is to use the included WP-CLI tool to flush your Redis cache:

    $ heroku run wp cache flush --app demo-stateless-wp
    Success: The cache was flushed.
