# anti-jndi
Fun things against the abuse of the recent CVE-2021-44228 (Log4Shell) vulnerability using common web servers.

Based on the post by @shipilev (https://gist.github.com/shipilev/92e709a868f3d328b6636e1bfc21cf09) I ported his example to Apache2. A coworker did it for Lighttpd. I decided to make our examples public for convenience.

# Idea
There are only few reasons to put the string "jndi:" in Request Headers, User Agents or somewhere else. Currently, I only know one single reason to do it: The exploitation of CVE-2021-44228.
While the world is hopefully busy patching every implementation of vulnerable Log4j versions, it could be reasonable to slow down attackers as much as possible. So what about serving them some gigabytes of nonsense while they try to exploit your services?</details>

# Disclaimer
The following code snippets don't protect your devices and services against the Log4Shell 0-Day! Please update vulnerable software or take it offline until there's a patch available! Only use this on servers without Log4j-enabled services!

# Preparation (adopted from @shipilev)
On Linux, create a file with some random HTML message. Please **don't** use the *LOL* from the example below as it makes it easier for the attacker to implement a generic filter to evade us. We're using the *pv* utility to display the progress of the file creation, you might have to install it via your favourite package manager or simply omit it.
```
$ awk 'BEGIN { for(c=0;c<10000000;c++) printf "<p>LOL</p>" }' > 100M.html
$ (for I in `seq 1 100`; do cat 100M.html; done) | pv | gzip -9 > 10G.boomgz
```
Assuming that your web server runs as *www-data*, let's create a directory owned by *www-data* so we don't have to upload a copy in every webroot that the web server is possibly serving:
```
mkdir /bombs
mv 10G.boomgz /bombs
chown -R www-data:www-data /bombs
```
Now for the fun part...
# Nginx
See https://gist.github.com/shipilev/92e709a868f3d328b6636e1bfc21cf09

All credits go to @shipilev
# Apache2
Enable the required modules
```
a2enmod rewrite headers ratelimit
```
Navigate to /etc/apache2/sites-enabled.
Open every host configuration file with your favourite editor and insert the following code snippet just above every line containing *</VirtualHost>*
```
RewriteEngine On
RewriteCond %{THE_REQUEST} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{QUERY_STRING} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{REQUEST_URI} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{HTTP_COOKIE} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{HTTP_HOST} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{REMOTE_HOST} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{REMOTE_USER} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{HTTP_USER_AGENT} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
RewriteCond %{HTTP_REFERER} "^.*jndi:.*$"
RewriteRule . /bombs/10G_lol.boomgz [L]
<Files ~ "\.boomgz$">
Header Set Expires "Sat, 1 Jan 2000 00:00:00 GMT"
Header Set Content-Encoding "gzip"
Header Set Content-Type "text/html"
SetOutputFilter RATE_LIMIT
SetEnv rate-limit 100
</Files>
<Directory /bombs>
allow from allow
Require all granted
</Directory>
```
You may wonder why there are so much more variables than in the initial example by @shipilev. I decided to try to catch as many locations as possible. This might interrupt services in some very rare cases so if you want to use the original set of checks, just comment out or delete the first 14 lines after the *RewriteEngine On* statement.
Now save the file(s) and reload the Apache2 configuration:
```
systemctl reload apache2
```
If everything went well you should not get an error message.
# Lighttpd
Coming soon
# Testing if everything works
You can check if you succeeded by connecting to your altered website via curl:
```
curl -s -L your-hostname -A "\${jndi:testing}" | pv > /dev/null
```
You should see a progress bar, showing around 100kb/s download speed. If you don't cancel it, it should be finished after some minutes, depending on what you used as the string in the initial HTML file. For me, it took around 5 minutes.
Now check what's happening on the client side:
```
curl -s --compressed -L your-hostname -A "\${jndi:testing}" | pv > /dev/null
```
See how it's not 100kb/s any more? Compression does its magic, it's some gigabytes at client side. So after downloading a pile of nonsense for some minutes, the attacker has some gigabytes lying around, given the data is stored for analysis. Who knows? :)
