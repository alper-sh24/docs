# Serving HTML Bombs to Exploiters

This is a clever "tarpit" defense strategy. The idea is to waste attackers' resources instead of just blocking them.

## The Bomb Generator

A collection of _contextual_ bombs can be generated with Python:

```py
#!/usr/bin/env python3
"""
Generate HTML zip bombs for honeypot defense.
These files are small when compressed but expand to gigabytes when decompressed.

Usage:
    python generate_bomb.py

This creates several .html.gz files that can be served to attackers.
"""

import gzip
import os

OUTPUT_DIR = "./bombs"


def generate_html_bomb(filename: str, size_mb: int = 10):
    """
    Generate a valid HTML file that contains a massive comment.
    When gzipped, identical repeated characters compress extremely well.

    A 10MB gzipped file can expand to ~10GB when decompressed.
    """
    # Calculate how many bytes we need (aiming for ~1GB decompressed per 1MB compressed)
    # Using zeros or repeated chars gives ~1000:1 compression ratio
    decompressed_size = size_mb * 1024 * 1024 * 1000  # Target decompressed size

    html_start = b'''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Loading...</title>
</head>
<body>
    <h1>Please wait...</h1>
    <p>Loading content...</p>
    <!--
'''

    html_end = b'''
    -->
    <script>console.log("loaded");</script>
</body>
</html>
'''

    os.makedirs(OUTPUT_DIR, exist_ok=True)
    filepath = os.path.join(OUTPUT_DIR, filename)

    print(f"Generating {filename}...")
    print(f"  Target decompressed size: ~{decompressed_size / (1024**3):.1f} GB")

    # Write directly to gzip to avoid memory issues
    with gzip.open(filepath, 'wb', compresslevel=9) as f:
        f.write(html_start)

        # Write in chunks to avoid memory issues
        # Using null bytes (\x00) for maximum compression
        chunk_size = 10 * 1024 * 1024  # 10MB chunks
        chunk = b'\x00' * chunk_size

        bytes_written = len(html_start)
        target = decompressed_size

        while bytes_written < target:
            f.write(chunk)
            bytes_written += chunk_size
            if bytes_written % (100 * 1024 * 1024) == 0:  # Progress every 100MB
                print(f"  Written {bytes_written / (1024**3):.1f} GB...")

        f.write(html_end)

    # Get actual file size
    actual_size = os.path.getsize(filepath)
    print(f"  Compressed file size: {actual_size / (1024**2):.2f} MB")
    print(f"  Compression ratio: ~{decompressed_size / actual_size:.0f}:1")
    print(f"  Saved to: {filepath}")
    print()


def generate_fake_files():
    """Generate various fake files that attackers commonly probe for."""

    # Different sizes for different scenarios
    # Smaller bombs for high-volume endpoints, larger for targeted attacks
    bombs = [
        ("wp-login.html.gz", 1),      # 1MB -> ~1GB (WordPress login)
        ("wp-admin.html.gz", 1),       # WordPress admin
        ("phpmyadmin.html.gz", 2),     # 2MB -> ~2GB (phpMyAdmin)
        ("admin.html.gz", 1),          # Generic admin
        ("config.html.gz", 5),         # 5MB -> ~5GB (config files - make it hurt)
        (".env.html.gz", 5),           # Environment files
        ("backup.html.gz", 10),        # 10MB -> ~10GB (backup probes - maximum pain)
        ("shell.html.gz", 10),         # Shell upload attempts
        ("default.html.gz", 1),        # Default bomb for unknown exploits
    ]

    print("=" * 50)
    print("HTML ZIP BOMB GENERATOR")
    print("=" * 50)
    print()
    print("WARNING: These files are designed to crash/slow down")
    print("malicious crawlers and vulnerability scanners.")
    print()

    for filename, size_mb in bombs:
        generate_html_bomb(filename, size_mb)

    print("=" * 50)
    print("GENERATION COMPLETE")
    print("=" * 50)
    print()
    print("Next steps:")
    print("")
    print("2. ")
    print("3. See honeypot.conf for Apache configuration")


if __name__ == "__main__":
    generate_fake_files()
```

You can use the smaller **bombs** for high-volume endpoints, and the larger ones for targeted attacks.
After you generate the bombs:

1. Upload them to the desired directory on your server. The folder name can be anything you like.
2. Configure Apache/Nginx to serve these files
3. Have fun!

## Apache Config

To serve the **bombs** with Apache rewrites, you must make some configurations. All examples below assume your **bombs** folder is named `honeypot`.

You can use this `honeypot.conf`:

```conf
# =============================================================================
# APACHE HONEYPOT CONFIGURATION - ZIP BOMB DEFENSE
# =============================================================================
#
# This configuration serves gzip bombs to attackers probing for common exploits.
# The bombs are small files (~1-10MB) that decompress to gigabytes, crashing
# or severely slowing down vulnerability scanners and malicious crawlers.
#
# INSTALLATION:
# 1. Generate bombs: python generate_bomb.py
# 2. Upload bombs directory to /var/www/honeypot/ (or your preferred location)
# 3. Include this file in your Apache config: Include /etc/apache2/conf-available/honeypot.conf
# 4. Enable required modules: a2enmod rewrite headers
# 5. Restart Apache: systemctl restart apache2
#
# =============================================================================

# Alias for the honeypot directory
Alias /honeypot /var/www/honeypot

<Directory /var/www/honeypot>
    Options -Indexes
    AllowOverride None
    Require all granted

    # Serve pre-compressed files directly WITHOUT decompressing
    # This is the key - Apache sends the .gz file with gzip encoding header
    <FilesMatch "\.gz$">
        Header set Content-Encoding gzip
        Header set Content-Type "text/html; charset=UTF-8"

        # Remove .gz from the served filename
        # So bomb.html.gz is served as bomb.html
        RemoveType .gz
        AddEncoding gzip .gz

        # Prevent caching (we want them to download it every time)
        Header set Cache-Control "no-store, no-cache, must-revalidate"
        Header set Pragma "no-cache"
    </FilesMatch>
</Directory>

# =============================================================================
# WORDPRESS EXPLOIT REDIRECTS
# =============================================================================
# These patterns catch WordPress vulnerability scanners

RewriteEngine On

# WordPress paths -> wp-login bomb
RewriteCond %{REQUEST_URI} ^.*(wp-login|wp-admin|wp-content|wp-includes|wp-config|wp-json|xmlrpc\.php|wp-cron|wlwmanifest).*$ [NC]
RewriteRule .* /honeypot/wp-login.html.gz [L]

# =============================================================================
# PHP/CMS EXPLOIT REDIRECTS
# =============================================================================

# phpMyAdmin probes
RewriteCond %{REQUEST_URI} ^.*(phpmyadmin|pma|myadmin|mysql|adminer).*$ [NC]
RewriteRule .* /honeypot/phpmyadmin.html.gz [L]

# PHP shell/backdoor probes
RewriteCond %{REQUEST_URI} ^.*(shell\.php|c99\.php|r57\.php|eval-stdin|php-cgi).*$ [NC]
RewriteRule .* /honeypot/shell.html.gz [L]

# phpinfo probes
RewriteCond %{REQUEST_URI} ^.*phpinfo\.php.*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# SENSITIVE FILE PROBES
# =============================================================================

# Environment and config files (maximum pain - 10GB bomb)
RewriteCond %{REQUEST_URI} ^.*(\.env|\.git/|\.gitignore|\.htaccess|\.htpasswd|\.ssh|\.aws).*$ [NC]
RewriteRule .* /honeypot/config.html.gz [L]

# Config file probes
RewriteCond %{REQUEST_URI} ^.*(config\.php|configuration\.php|settings\.php|database\.yml|secrets\.yml|credentials).*$ [NC]
RewriteRule .* /honeypot/config.html.gz [L]

# Backup file probes (maximum pain)
RewriteCond %{REQUEST_URI} ^.*(\.bak|\.backup|\.old|\.sql|\.tar\.gz|\.zip|dump\.sql|backup\.sql)$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# Log file probes
RewriteCond %{REQUEST_URI} ^.*(debug\.log|error\.log|access\.log).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# ADMIN PANEL PROBES
# =============================================================================

# Generic admin probes (but NOT your legitimate /admin path!)
# Adjust this pattern based on your actual admin URL structure
RewriteCond %{REQUEST_URI} ^.*(administrator|/admin\.php|/login\.php|/manager/|cpanel|plesk|webmail|roundcube).*$ [NC]
RewriteRule .* /honeypot/admin.html.gz [L]

# Server info probes
RewriteCond %{REQUEST_URI} ^.*(server-status|server-info|cgi-bin/).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# VULNERABILITY SCANNER PATHS
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(actuator|swagger|graphql|/debug/|/trace/|/console/|solr|jenkins|jmx-console).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# OTHER CMS PROBES
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(joomla|drupal|magento|typo3|contao|concrete5|prestashop|opencart|shopware|umbraco|sitecore|kentico).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# PATH TRAVERSAL ATTEMPTS
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(\.\.\/|\.\.%2f|%2e%2e|etc/passwd|etc/shadow|proc/self|boot\.ini|win\.ini).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# =============================================================================
# WEB SHELL PARAMETERS (query string attacks)
# =============================================================================

# Catch shell commands in query strings
RewriteCond %{QUERY_STRING} ^.*(cmd=|command=|exec=|execute=|system=|passthru=|shell_exec=|base64_decode|eval\().*$ [NC]
RewriteRule .* /honeypot/shell.html.gz [L]

# =============================================================================
# LOG4J / LOG4SHELL
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(\$\{jndi:|%24%7bjndi|\$\{env:|\$\{lower:|\$\{upper:).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# Also check query strings and headers for Log4j
RewriteCond %{QUERY_STRING} ^.*(\$\{jndi:|%24%7bjndi).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# =============================================================================
# OPTIONAL: BLOCK KNOWN SCANNER USER AGENTS
# =============================================================================
# Uncomment these to also target known vulnerability scanners by User-Agent

# RewriteCond %{HTTP_USER_AGENT} ^.*(nmap|nikto|sqlmap|acunetix|nessus|burp|masscan|zgrab).*$ [NC]
# RewriteRule .* /honeypot/backup.html.gz [L]

# =============================================================================
# LOGGING (Optional - track honeypot hits)
# =============================================================================
# Add to your VirtualHost to log honeypot access separately:
#
# SetEnvIf Request_URI "^/honeypot" honeypot_request
# CustomLog ${APACHE_LOG_DIR}/honeypot.log combined env=honeypot_request
```

Or, if you have a `.htaccess` file that you would like to extend, you can append this to the _root_ `.htaccess`:

```conf
# =============================================================================
# HONEYPOT .HTACCESS - ZIP BOMB DEFENSE (Shared Hosting Version)
# =============================================================================
#
# INSTALLATION:
# 1. Generate bombs: python generate_bomb.py
# 2. Upload the /honeypot/ folder to your public_html directory
# 3. Merge this into your existing .htaccess (or replace if none exists)
# 4. Rename this file to .htaccess
#
# STRUCTURE AFTER SETUP:
#   public_html/
#   ├── .htaccess          (this file)
#   ├── honeypot/
#   │   ├── .htaccess      (see bottom of this file)
#   │   ├── wp-login.html.gz
#   │   ├── phpmyadmin.html.gz
#   │   ├── shell.html.gz
#   │   ├── config.html.gz
#   │   ├── backup.html.gz
#   │   ├── admin.html.gz
#   │   └── default.html.gz
#   └── (your normal site files)
#
# =============================================================================

RewriteEngine On

# =============================================================================
# WORDPRESS EXPLOIT REDIRECTS
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(wp-login|wp-admin|wp-content|wp-includes|wp-config|wp-json|xmlrpc\.php|wp-cron|wlwmanifest).*$ [NC]
RewriteRule .* /honeypot/wp-login.html.gz [L]

# =============================================================================
# PHP/CMS EXPLOIT REDIRECTS
# =============================================================================

# phpMyAdmin probes
RewriteCond %{REQUEST_URI} ^.*(phpmyadmin|pma|myadmin|mysql|adminer).*$ [NC]
RewriteRule .* /honeypot/phpmyadmin.html.gz [L]

# PHP shell/backdoor probes
RewriteCond %{REQUEST_URI} ^.*(shell\.php|c99\.php|r57\.php|eval-stdin|php-cgi).*$ [NC]
RewriteRule .* /honeypot/shell.html.gz [L]

# phpinfo probes
RewriteCond %{REQUEST_URI} ^.*phpinfo\.php.*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# SENSITIVE FILE PROBES
# =============================================================================

# Environment and config files (maximum pain - 10GB bomb)
RewriteCond %{REQUEST_URI} ^.*(\.env|\.git/|\.gitignore|\.htpasswd|\.ssh|\.aws).*$ [NC]
RewriteRule .* /honeypot/config.html.gz [L]

# Config file probes
RewriteCond %{REQUEST_URI} ^.*(config\.php|configuration\.php|settings\.php|database\.yml|secrets\.yml|credentials).*$ [NC]
RewriteRule .* /honeypot/config.html.gz [L]

# Backup file probes (maximum pain)
RewriteCond %{REQUEST_URI} ^.*(\.bak|\.backup|\.old|\.sql|\.tar\.gz|\.zip|dump\.sql|backup\.sql)$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# Log file probes
RewriteCond %{REQUEST_URI} ^.*(debug\.log|error\.log|access\.log).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# ADMIN PANEL PROBES
# =============================================================================

# Generic admin probes
# NOTE: Adjust if you have a legitimate /admin path!
RewriteCond %{REQUEST_URI} ^.*(administrator|/admin\.php|/login\.php|/manager/|cpanel|plesk|webmail|roundcube).*$ [NC]
RewriteRule .* /honeypot/admin.html.gz [L]

# Server info probes
RewriteCond %{REQUEST_URI} ^.*(server-status|server-info|cgi-bin/).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# VULNERABILITY SCANNER PATHS
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(actuator|swagger|graphql|/debug/|/trace/|/console/|solr|jenkins|jmx-console).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# OTHER CMS PROBES
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(joomla|drupal|magento|typo3|contao|concrete5|prestashop|opencart|shopware|umbraco|sitecore|kentico).*$ [NC]
RewriteRule .* /honeypot/default.html.gz [L]

# =============================================================================
# PATH TRAVERSAL ATTEMPTS
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(\.\.\/|\.\.%2f|%2e%2e|etc/passwd|etc/shadow|proc/self|boot\.ini|win\.ini).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

# =============================================================================
# WEB SHELL PARAMETERS (query string attacks)
# =============================================================================

RewriteCond %{QUERY_STRING} ^.*(cmd=|command=|exec=|execute=|system=|passthru=|shell_exec=|base64_decode|eval\().*$ [NC]
RewriteRule .* /honeypot/shell.html.gz [L]

# =============================================================================
# LOG4J / LOG4SHELL
# =============================================================================

RewriteCond %{REQUEST_URI} ^.*(\$\{jndi:|%24%7bjndi|\$\{env:|\$\{lower:|\$\{upper:).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]

RewriteCond %{QUERY_STRING} ^.*(\$\{jndi:|%24%7bjndi).*$ [NC]
RewriteRule .* /honeypot/backup.html.gz [L]


# =============================================================================
# =============================================================================
#
# HONEYPOT FOLDER .HTACCESS
# --------------------------
# Create a SEPARATE .htaccess file inside /honeypot/ with this content:
#
# <FilesMatch "\.gz$">
#     Header set Content-Encoding gzip
#     Header set Content-Type "text/html; charset=UTF-8"
#     RemoveType .gz
#     AddEncoding gzip .gz
#     Header set Cache-Control "no-store, no-cache, must-revalidate"
#     Header set Pragma "no-cache"
# </FilesMatch>
#
# Options -Indexes
#
# =============================================================================
# =============================================================================
```

And, then you should create another `.htaccess` inside the folder which you copied the **bombs**:

```conf
# =============================================================================
# HONEYPOT FOLDER .HTACCESS
# =============================================================================
# Place this file as .htaccess inside your /<honeypot>/ directory
#
# This tells Apache to serve the .gz files with proper gzip headers
# so the browser/scanner decompresses them (and explodes)
# =============================================================================

# Serve .gz files with gzip encoding
<FilesMatch "\.gz$">
    Header set Content-Encoding gzip
    Header set Content-Type "text/html; charset=UTF-8"
    RemoveType .gz
    AddEncoding gzip .gz
    Header set Cache-Control "no-store, no-cache, must-revalidate"
    Header set Pragma "no-cache"
</FilesMatch>

# Prevent directory listing
Options -Indexes

# Deny access to this .htaccess file itself
<Files .htaccess>
    Require all denied
</Files>
```

---

Alternatively, if you use Nginx, you can use this configuration:

```
# =============================================================================
# NGINX HONEYPOT CONFIGURATION - ZIP BOMB DEFENSE
# =============================================================================
#
# This configuration serves gzip bombs to attackers probing for common exploits.
# Include this in your server block or http block.
#
# INSTALLATION:
# 1. Generate bombs: python generate_bomb.py
# 2. Upload bombs directory to /var/www/honeypot/
# 3. Include this file: include /etc/nginx/conf.d/honeypot.conf;
# 4. Test config: nginx -t
# 5. Reload: systemctl reload nginx
#
# =============================================================================

# Map to store the bomb file to serve based on attack type
map $request_uri $honeypot_bomb {
    default "";

    # WordPress probes
    ~*wp-login          /var/www/honeypot/wp-login.html.gz;
    ~*wp-admin          /var/www/honeypot/wp-admin.html.gz;
    ~*wp-content        /var/www/honeypot/wp-login.html.gz;
    ~*wp-includes       /var/www/honeypot/wp-login.html.gz;
    ~*wp-config         /var/www/honeypot/config.html.gz;
    ~*wp-json           /var/www/honeypot/wp-login.html.gz;
    ~*xmlrpc\.php       /var/www/honeypot/wp-login.html.gz;
    ~*wp-cron           /var/www/honeypot/wp-login.html.gz;
    ~*wlwmanifest       /var/www/honeypot/wp-login.html.gz;

    # phpMyAdmin probes
    ~*phpmyadmin        /var/www/honeypot/phpmyadmin.html.gz;
    ~*pma/              /var/www/honeypot/phpmyadmin.html.gz;
    ~*myadmin           /var/www/honeypot/phpmyadmin.html.gz;
    ~*adminer           /var/www/honeypot/phpmyadmin.html.gz;

    # PHP shell probes
    ~*shell\.php        /var/www/honeypot/shell.html.gz;
    ~*c99\.php          /var/www/honeypot/shell.html.gz;
    ~*r57\.php          /var/www/honeypot/shell.html.gz;
    ~*eval-stdin        /var/www/honeypot/shell.html.gz;
    ~*phpinfo\.php      /var/www/honeypot/default.html.gz;

    # Sensitive files (big bombs)
    ~*\.env             /var/www/honeypot/config.html.gz;
    ~*\.git/            /var/www/honeypot/config.html.gz;
    ~*\.gitignore       /var/www/honeypot/config.html.gz;
    ~*\.htaccess        /var/www/honeypot/config.html.gz;
    ~*\.htpasswd        /var/www/honeypot/config.html.gz;
    ~*\.ssh             /var/www/honeypot/config.html.gz;
    ~*\.aws             /var/www/honeypot/config.html.gz;
    ~*config\.php       /var/www/honeypot/config.html.gz;
    ~*database\.yml     /var/www/honeypot/config.html.gz;
    ~*secrets\.yml      /var/www/honeypot/config.html.gz;

    # Backup files (maximum pain)
    ~*\.bak$            /var/www/honeypot/backup.html.gz;
    ~*\.backup$         /var/www/honeypot/backup.html.gz;
    ~*\.old$            /var/www/honeypot/backup.html.gz;
    ~*\.sql$            /var/www/honeypot/backup.html.gz;
    ~*dump\.sql         /var/www/honeypot/backup.html.gz;
    ~*backup\.sql       /var/www/honeypot/backup.html.gz;

    # Admin probes
    ~*/administrator/   /var/www/honeypot/admin.html.gz;
    ~*/admin\.php       /var/www/honeypot/admin.html.gz;
    ~*/login\.php       /var/www/honeypot/admin.html.gz;
    ~*/manager/         /var/www/honeypot/admin.html.gz;
    ~*cpanel            /var/www/honeypot/admin.html.gz;
    ~*plesk             /var/www/honeypot/admin.html.gz;
    ~*webmail           /var/www/honeypot/admin.html.gz;

    # Vulnerability scanner paths
    ~*actuator          /var/www/honeypot/default.html.gz;
    ~*swagger           /var/www/honeypot/default.html.gz;
    ~*graphql           /var/www/honeypot/default.html.gz;
    ~*/debug/           /var/www/honeypot/default.html.gz;
    ~*/trace/           /var/www/honeypot/default.html.gz;
    ~*/console/         /var/www/honeypot/default.html.gz;
    ~*jenkins           /var/www/honeypot/default.html.gz;
    ~*solr              /var/www/honeypot/default.html.gz;

    # Other CMS probes
    ~*joomla            /var/www/honeypot/default.html.gz;
    ~*drupal            /var/www/honeypot/default.html.gz;
    ~*magento           /var/www/honeypot/default.html.gz;
    ~*typo3             /var/www/honeypot/default.html.gz;
    ~*prestashop        /var/www/honeypot/default.html.gz;
    ~*opencart          /var/www/honeypot/default.html.gz;
    ~*shopware          /var/www/honeypot/default.html.gz;

    # Path traversal
    ~*\.\.\/            /var/www/honeypot/backup.html.gz;
    ~*etc/passwd        /var/www/honeypot/backup.html.gz;
    ~*proc/self         /var/www/honeypot/backup.html.gz;

    # Log4j
    ~*\$\{jndi          /var/www/honeypot/backup.html.gz;
    ~*%24%7bjndi        /var/www/honeypot/backup.html.gz;
}

# Also check query strings for shell commands
map $query_string $honeypot_query_bomb {
    default "";
    ~*cmd=              /var/www/honeypot/shell.html.gz;
    ~*command=          /var/www/honeypot/shell.html.gz;
    ~*exec=             /var/www/honeypot/shell.html.gz;
    ~*system=           /var/www/honeypot/shell.html.gz;
    ~*passthru=         /var/www/honeypot/shell.html.gz;
    ~*base64_decode     /var/www/honeypot/shell.html.gz;
    ~*\$\{jndi          /var/www/honeypot/backup.html.gz;
}

# =============================================================================
# ADD THIS TO YOUR SERVER BLOCK
# =============================================================================

# Uncomment and add inside your server { } block:

# # Honeypot - serve zip bombs to attackers
# set $bomb "";
# if ($honeypot_bomb != "") {
#     set $bomb $honeypot_bomb;
# }
# if ($honeypot_query_bomb != "") {
#     set $bomb $honeypot_query_bomb;
# }
#
# # If we have a bomb to serve, serve it
# if ($bomb != "") {
#     rewrite ^ /internal_honeypot last;
# }
#
# # Internal location for serving bombs
# location = /internal_honeypot {
#     internal;
#
#     # Serve the pre-compressed file directly
#     gzip off;  # Don't double-compress!
#     gzip_static off;
#
#     # Set headers to make client decompress it
#     add_header Content-Encoding gzip always;
#     add_header Content-Type "text/html; charset=UTF-8" always;
#     add_header Cache-Control "no-store, no-cache" always;
#
#     # Serve the bomb file
#     alias $bomb;
# }

# =============================================================================
# ALTERNATIVE: SIMPLER LOCATION-BASED APPROACH
# =============================================================================
# If the map approach is too complex, use explicit locations:

# location ~* (wp-login|wp-admin|wp-content|wp-includes) {
#     gzip off;
#     add_header Content-Encoding gzip always;
#     add_header Content-Type "text/html" always;
#     alias /var/www/honeypot/wp-login.html.gz;
# }
#
# location ~* (phpmyadmin|pma/|myadmin) {
#     gzip off;
#     add_header Content-Encoding gzip always;
#     add_header Content-Type "text/html" always;
#     alias /var/www/honeypot/phpmyadmin.html.gz;
# }
#
# location ~* (\.env|\.git/|\.htaccess|\.htpasswd) {
#     gzip off;
#     add_header Content-Encoding gzip always;
#     add_header Content-Type "text/html" always;
#     alias /var/www/honeypot/config.html.gz;
# }
#
# location ~* (shell\.php|c99\.php|r57\.php) {
#     gzip off;
#     add_header Content-Encoding gzip always;
#     add_header Content-Type "text/html" always;
#     alias /var/www/honeypot/shell.html.gz;
# }

# =============================================================================
# LOGGING HONEYPOT HITS
# =============================================================================
# Add this log format to track attackers:
#
# log_format honeypot '$remote_addr - $time_iso8601 - "$request" - "$http_user_agent"';
#
# In your server block:
# access_log /var/log/nginx/honeypot.log honeypot if=$bomb;
```
