# Magento 2 permission Guide

This guide is intended to show best practices for Magento 2 permissions. Ensuring that required directories are accessible whilst restricting access to others.

```eval_rst
.. seealso::
   Note:
     The following is for UKFast Magento 2 optimized stacks, which uses NGINX - this guide is for NGINX only.
     If your configuration is bespoke or does not use our native stack we would recommend proceeding with caution.

     In the case that you have any queries please consult UKFast support.
```

First, we need to establish the root directory for your Magento 2 instance. Typically this can be viewed in the NGINX configuration file located under:
/etc/nginx/conf.d/

```bash
grep root /etc/nginx/conf.d/mage.ukast.co.uk.conf
  root /var/www/vhosts/mage.ukast.co.uk/htdocs;
```

Next, we need to verify which user and group should be used. This is important as if the domain is a Magento multi-store, typically the owner and group would be the same for each virtual host as the document root is shared.

In this case, PHP-FPM is utilized - this is configured as an upstream within Nginx -

```bash
grep ".sock;" /etc/nginx/conf.d/mage.ukast.co.uk.conf
server unix:/var/run/php-fpm-mageukastcouk;

grep php-fpm-mageukastcouk /etc/php-fpm.d/*.conf
/etc/php-fpm.d/mageukastcouk.conf:listen = '/var/run/php-fpm-php-fpm-mageukastcouk'

egrep "^user|^group" /etc/php-fpm.d/mageukastcouk.conf
user = mage.ukast.co.uk
group = mage.ukast.co.uk
```

Using all the information provided above, we can then input this into the commands for changing the document roots permissions.

```eval_rst
.. warning::
   **Please ensure the getfacl command is run first, this means that we can revert your permissions back if required.**
```

```bash
getfacl -R /var/www/vhosts/mage.ukast.co.uk/htdocs > /var/www/vhosts/mage.ukast.co.uk/mage.ukast.co.uk-$(date +"%Y%m%d").acl
```

If this needs to be restored, the acl file can be used to restore the original permissions:

```bash
setfacl --restore=/var/www/vhosts/mage.ukast.co.uk/mage.ukast.co.uk-$(date +"%Y%m%d").acl
```

We can now proceed with updating the permissions. Please remember to change the root directory and user in accordance with the output of the previous commands.
```bash
chown -R exampleuser:examplegroup <your Magento install dir>

find <your Magento install dir> -type f -exec chmod 440 {} \;
find <your Magento install dir> -type d -exec chmod 550 {} \;
find <your Magento install dir>/var/ -type f -exec chmod 660 {} \;
find <your Magento install dir>/var/ -type d -exec chmod 770 {} \;
find <your Magento install dir>/pub/media/ -type f -exec chmod 660 {} \;
find <your Magento install dir>/pub/media/ -type d -exec chmod 770 {} \;
```

If not already added, you can add Nginx to the group used by PHP-FPM - here is the command to do so - this only needs to be performed once
```bash
usermod -a -G mage.ukast.co.uk nginx
```

To verify that this has been added to the group
```bash
id nginx
uid=10(nginx) gid=10(nginx) groups=11(mage.ukast.co.uk)
```
Note that the Above permissions changes are for a production site, as such write access is restricted for improved site security.

When deploying, you will need to add write permissions.

However, As of Magento 2.2.0 the following has been changed:
----
* var/generation  --> generated/code
* var/di          --> generated/metadata

More information here (note the difference between the two):
 https://devdocs.magento.com/guides/v2.1/howdoi/php/php_clear-dirs.html
 https://devdocs.magento.com/guides/v2.2/howdoi/php/php_clear-dirs.html

Therefore before running this, you need to check the Magento version.
```bash
su - exampleuser
cd <your Magento install dir>
php bin/magento --version
```

## Magento 2.1
(versions lesser than or equal to.)

Removal of write access as Magento recommends in production:
```bash
cd <your Magento install dir> && find app/code lib pub/static app/etc var/generation var/di var/view_preprocessed vendor \( -type d -or -type f \) -exec chmod g-w {} + && chmod o-rwx app/etc/env.php
```
Add Write access when in development mode or when changes are required:
```bash
cd <your Magento install dir> && find app/code lib var pub/static pub/media vendor app/etc \( -type d -or -type f \) -exec chmod g+w {} + && chmod o+rwx app/etc/env.php && chmod u+x bin/magento
```
## Magento 2.2
(versions greater than or equal to.)
Removal of write access as Magento recommends in production:
```bash
cd <your Magento install dir> && find app/code lib pub/static app/etc generated/code generated/metadata var/view_preprocessed vendor \( -type d -or -type f \) -exec chmod g-w {} + && chmod o-rwx app/etc/env.php
```

Add Write access when in development mode or when changes are required:
```bash
cd <your Magento install dir> && find app/code lib var generated/code generated/metadata pub/static pub/media vendor app/etc \( -type d -or -type f \) -exec chmod g+w {} + && chmod o+rwx app/etc/env.php && chmod u+x bin/magento
```

Now the permissions for the site should be correct. However, if you are unsure about any of the above then please do hesitate to get in touch with UKFast support and we can assist you further.
