---
layout: default
parent: Contents
date: 2025-07-06
nav_exclude: true
---

## Tidying Up Your MediaWiki URLs: A Guide to Hiding "index.php"
- TOC
{:toc}

A cleaner, more user-friendly URL structure for your MediaWiki installation is easily achievable by hiding the "/index.php" portion of your document addresses. This process, often referred to as creating "short URLs," not only enhances the aesthetics of your links but can also improve search engine optimization. This guide will walk you through the necessary steps to achieve this for the popular web servers, Apache and Nginx.

The core of this process involves two key modifications: first, adjusting your MediaWiki's `LocalSettings.php` file to generate the desired URL format, and second, configuring your web server to correctly interpret these new URLs and route them to the appropriate MediaWiki script.

### Step 1: Modifying `LocalSettings.php`

The first step is to inform MediaWiki about your new URL structure. This is done by adding or modifying a few lines in your `LocalSettings.php` file, which is located in the root directory of your MediaWiki installation.

Open your `LocalSettings.php` file and add the following line:

```php
$wgArticlePath = "/wiki/$1";
```

This line tells MediaWiki to format the URLs for your articles as `https://yourdomain.com/wiki/Article_Name`. You can replace `/wiki` with any other path you prefer, but it is a widely used and recommended convention.

It is also good practice to ensure the following setting is present, which is the default in modern MediaWiki versions:

```php
$wgUsePathInfo = true;
```

This setting enables the use of "path info" in URLs, which is essential for the short URL functionality to work correctly.

### Step 2: Configuring Your Web Server

After updating `LocalSettings.php`, you need to configure your web server to handle the new "pretty" URLs. The server needs to understand that a request for `/wiki/Article_Name` should actually be processed by the `index.php` script.

#### For Apache Users

If you are using the Apache web server, the most common way to implement URL rewriting is by using a `.htaccess` file.

1.  **Enable `mod_rewrite`:** Ensure that the `mod_rewrite` module is enabled in your Apache configuration. You can usually do this by running `sudo a2enmod rewrite` on Debian-based systems or by uncommenting the corresponding line in your `httpd.conf` file.

2.  **Create or edit your `.htaccess` file:** In the root directory of your MediaWiki installation (the same directory as `LocalSettings.php`), create or edit the `.htaccess` file and add the following lines:

    ```apache
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^/?wiki/(.*)$ /index.php?title=$1 [L,QSA]
    ```

    This configuration does the following:
    * `RewriteEngine On`: Enables the rewriting engine.
    * `RewriteCond %{REQUEST_FILENAME} !-f` and `RewriteCond %{REQUEST_FILENAME} !-d`: These conditions prevent the rewrite rule from being applied to existing files or directories. This is important to ensure that direct requests to files like images or CSS are not rewritten.
    * `RewriteRule ^/?wiki/(.*)$ /index.php?title=$1 [L,QSA]`: This is the main rule. It captures the article title from the URL and passes it as a `title` parameter to `index.php`. The `[L]` flag tells Apache to stop processing further rules, and `[QSA]` appends any existing query string to the rewritten URL.

**Important Note on Directory Structure:** It is highly recommended that your MediaWiki installation itself is not in a directory named `/wiki`. A common practice is to install MediaWiki in a directory like `/w` and then use `/wiki` as the virtual path for your articles. This prevents potential conflicts between the physical directory and the virtual path.

#### For Nginx Users

For those using the Nginx web server, you will need to edit your site's configuration file, which is typically located in `/etc/nginx/sites-available/`.

Within the `server` block of your site's configuration, add the following `location` block:

```nginx
location /wiki/ {
    rewrite ^/wiki/(.*)$ /index.php?title=$1;
}

location / {
    try_files $uri $uri/ @rewrite;
}

location @rewrite {
    rewrite ^/(.*)$ /index.php?title=$1;
}
```

This Nginx configuration achieves the same outcome as the Apache `.htaccess` rules. It defines a specific location block for your `/wiki/` path and rewrites the URL to be handled by `index.php`. The additional `location` blocks ensure that other requests are handled correctly.

After saving the changes to your Nginx configuration, you will need to restart the Nginx service for the changes to take effect. You can usually do this with the command `sudo systemctl restart nginx`.

### Troubleshooting Common Issues

If you encounter problems after implementing these changes, here are a few common issues and their solutions:

* **404 Not Found Errors:** If you are seeing "404 Not Found" errors for your wiki pages, it is likely that the web server rewrite rules are not being processed correctly.
    * **Apache:** Double-check that `mod_rewrite` is enabled and that your `.htaccess` file is in the correct directory and has the correct permissions to be read by the web server. Also, ensure that your main Apache configuration allows for overrides from `.htaccess` files (the `AllowOverride` directive should be set to `All` for your web root).
    * **Nginx:** Verify that you have reloaded or restarted the Nginx service after modifying the configuration. Check the Nginx error logs for any syntax errors in your configuration file.

* **Broken Links and Missing Styles:** If the page loads but without any styling or images, it could be that the rewrite rules are incorrectly redirecting requests for these assets. The `RewriteCond` directives in the Apache configuration and the `try_files` directive in the Nginx configuration are designed to prevent this. Ensure they are correctly implemented.

* **Redirect Loops:** If you find yourself in a redirect loop, carefully review your `LocalSettings.php` and your web server configuration for any conflicting rules or incorrect paths.

By following these steps, you can successfully remove `/index.php` from your MediaWiki URLs, resulting in a cleaner and more professional-looking wiki. For more advanced configurations or if you encounter persistent issues, the official MediaWiki documentation on "Short URLs" is an excellent and comprehensive resource.