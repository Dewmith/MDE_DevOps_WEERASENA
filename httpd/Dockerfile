# Use the official Apache HTTP Server image
FROM httpd:2.4

# Copy the landing page to the Apache server's document root
COPY index.html /usr/local/apache2/htdocs/

# Copy the custom configuration file
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
