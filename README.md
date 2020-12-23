## Ansible-wordpress
 This ansible playbooks used to install wordpress on amazon linux and ubuntu. This will also install lamp on the server.
 
 ### playbook for amazon linux
 ```
 ---
- name: "installing Lampstack on amazon linux"
  become: true
  hosts: amazon
  vars_files:
    - httpd.vars
    - mariadb.vars
    - wordpress.vars
  tasks:

    - name: "Installing apache and php"
      yum:
        name:
          - httpd
          - php
          - php-mysql
        state: present

    - name: "Copying httpd.conf file"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"

    - name: "Creating virtualhost"
      template:
        src: virtualhost.conf.tmpl
        dest: "/etc/httpd/conf.d/{{httpd_domain}}.conf"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"

    - name: "Creating documentroot"
      file:
        path: "/var/www/html/{{ httpd_domain }}"
        state: directory
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"

    - name: "Copying website content"
      copy:
        src: "{{ item }}"
        dest: "/var/www/html/{{ httpd_domain }}"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"
      with_items:
        - test.html
        - test.php

    - name: "Restarting/Enabling Apache"
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Mariadb-server installation"
      yum:
        name:
          - mariadb-server
          - MySQL-python
        state: present

    - name: "Mariadb-server restart/enable"
      service:
        name: mariadb
        state: restarted
        enabled: true

    - name: "Mariadb-server - Reset root password"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        name: "root"
        password: "{{ mariadb_root }}"
        host_all: true

    - name: "Mariadb-server - Removing anonymous users"
      mysql_user:
        login_user: "root"
        login_password: "{{ mariadb_root }}"
        name: ""
        state: absent
        host_all: true

    - name: "Mariadb-server - Creating extra Database"
      mysql_db:
        login_user: "root"
        login_password: "{{ mariadb_root }}"
        name: "{{ mariadb_database }}"
        state: present

    - name: "Mariadb-server - Creating extra user"
      mysql_user:
        login_user: "root"
        login_password: "{{ mariadb_root }}"
        name: "{{ mariadb_user }}"
        password: "{{ mariadb_password }}"
        state: present
        priv: "{{ mariadb_database }}.*:ALL"

    - name: "Wordpress - downloading archive files"
      get_url:
        url: "{{ wordpress_url }}"
        dest: /tmp/wordpress.tar.gz

    - name: "wordpress - extracting archive file"
      unarchive:
        remote_src: true
        src: /tmp/wordpress.tar.gz
        dest: /tmp/
        
    - name: "wordpress - copying file to documentroot"
      copy:
        src: /tmp/wordpress/
        dest: "/var/www/html/{{ httpd_domain }}"
        remote_src: true
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"

    - name: "wordpress - creating wp-config.php"
      template:
        src: wp-config.php.tmpl
        dest: "/var/www/html/{{ httpd_domain }}/wp-config.php"
        owner: "{{ httpd_owner }}"
        group: "{{ httpd_group }}"

    - name: "post-restart"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb

    - name: "post-cleanup"
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /tmp/wordpress
        - /tmp/wordpress.tar.gz
```
### Vars files
vim httpd.vars
```
---
httpd_owner: "apache"
httpd_group: "apache"
httpd_port: "8080"
httpd_domain: "www.linuxkernal.com"
```
vim mariadb.vars
```
---
mariadb_root: "mypassword"
mariadb_user: "wpuser"
mariadb_password: "wordpress123"
mariadb_database: "wordpress"
```
vim wordpress.vars
```
---
wordpress_url: "https://wordpress.org/wordpress-4.7.7.tar.gz"
mariadb_root: "mypassword"
mariadb_user: "wpuser"
mariadb_password: "wordpress123"
mariadb_database: "wordpress"
```
### Configuration files
vim virtualhost.conf.tmpl
```
<virtualhost *:{{ httpd_port }}>

        servername {{ httpd_domain }}
        documentroot /var/www/html/{{ httpd_domain }}
        directoryindex index.php index.html test.php test.html

</virtualhost>
```
vim wp-config.php.tmpl
```
<?php

/** The name of the database for WordPress */
define( 'DB_NAME', "{{mysql_database}}" );

/** MySQL database username */
define( 'DB_USER', "{{mysql_user}}" );

/** MySQL database password */
define( 'DB_PASSWORD', "{{mysql_password}}" );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );


define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );


$table_prefix = 'wp_';


define( 'WP_DEBUG', false );


if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', dirname( __FILE__ ) . '/' );
}

require_once( ABSPATH . 'wp-settings.php' );
```

### playbook for ubuntu 
```
---
- name: "install wordpress on ububtu"
  become: true
  hosts: ubuntu
  vars_files:
    - varubuntu.vars

  tasks:
    - name: "Install LAMP Packages"
      apt: name=apache2,mysql-server,python3-pymysql,php,php-mysql,libapache2-mod-php update_cache=yes state=latest

    - name: "Install PHP Extensions"
      apt: name={{ item }} update_cache=yes state=latest
      loop: "{{ php_modules }}"


    - name: "creating documentroot"
      file:
        path: "/var/www/{{ httpd_domain }}"
        state: directory
        owner: "www-data"
        group: "www-data"
        mode: '0755'

    - name: "Copying website content"
      copy:
        src: "{{ item }}"
        dest: "/var/www/{{ httpd_domain }}"
      with_items:
        - test.html
        - test.php

    - name: "copying virtual host"
      template:
        src: httpdconf.conf.tmpl
        dest: "/etc/apache2/sites-available/{{ httpd_domain }}.conf"

    - name: "Enable new site"
      shell: /usr/sbin/a2ensite {{ httpd_domain }}.conf

    - name: "Disable default Apache site"
      shell: /usr/sbin/a2dissite 000-default.conf

    - name: "UFW - Allow HTTP on port { httpd_port }}"
      ufw:
        rule: allow
        port: "{{ httpd_port }}"
        proto: tcp

    - name: "Restarting/Enabling Apache"
      service:
        name: apache2
        state: restarted
        enabled: true
        
    - name: "mysql-server - Reset root password"
      ignore_errors: true
      mysql_user:
        name: root
        password: "{{ mysql_root }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock
        host_all: true

    - name: "mysql-server - Removing anonymous users"
      mysql_user:
        name: ''
        host_all: yes
        state: absent
        login_user: root
        login_password: "{{ mysql_root }}"

    - name: "mysql-server - Creating extra Database"
      mysql_db:
        login_user: "root"
        login_password: "{{ mysql_root }}"
        name: "{{ mysql_database }}"
        state: present

    - name: "mysql-server - Creating extra user"
      mysql_user:
        login_user: "root"
        login_password: "{{ mysql_root }}"
        name: "{{ mysql_user }}"
        password: "{{ mysql_password }}"
        state: present
        priv: "{{ mysql_database }}.*:ALL"

    - name: "Wordpress - downloading archive files"
      get_url:
        url: "{{ wordpress_url }}"
        dest: /tmp/wordpress.tar.gz

    - name: "wordpress - extracting archive file"
      unarchive:
        remote_src: true
        src: /tmp/wordpress.tar.gz
        dest: /tmp/

    - name: "wordpress - copying file to documentroot"
      copy:
        src: /tmp/wordpress/
        dest: "/var/www/{{ httpd_domain }}"
        remote_src: true
        owner: "www-data"
        group: "www-data"
        
    - name: Set up wp-config
      template:
        src: ubuntu-wp-config.php.tmpl
        dest: "/var/www/{{ httpd_domain }}wp-config.php"

    - name: "post-restart"
      service:
        name: "{{ item }}"
        state: restarted
        enabled: true
      with_items:
        - apache2
        - mysql.service
```
### Vars file
vim varubuntu.vars
```
php_modules: [ 'php-curl', 'php-gd', 'php-mbstring', 'php-xml', 'php-xmlrpc', 'php-soap', 'php-intl', 'php-zip' ]
httpd_port: "80"
httpd_domain: "www.linuxkernal.com"
wordpress_url: "https://wordpress.org/wordpress-4.7.7.tar.gz"
mysql_root: "mypassword"
mysql_user: "wpuser"
mysql_password: "wordpress123"
mysql_database: "wordpress"
```
### Configuration file
vim httpdconf.conf.tmpl
```
<VirtualHost *:{{ httpd_port }}>
    ServerAdmin webmaster@localhost
    ServerName {{ httpd_domain }}
    ServerAlias {{ httpd_domain }}
    DocumentRoot /var/www/{{ httpd_domain }}
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
vim ubuntu-wp-config.php.tmpl
```
<?php

/** The name of the database for WordPress */
define( 'DB_NAME', "{{mysql_database}}" );

/** MySQL database username */
define( 'DB_USER', "{{mysql_user}}" );

/** MySQL database password */
define( 'DB_PASSWORD', "{{mysql_password}}" );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );


define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );


$table_prefix = 'wp_';


define( 'WP_DEBUG', false );


if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', dirname( __FILE__ ) . '/' );
}

require_once( ABSPATH . 'wp-settings.php' );
```
