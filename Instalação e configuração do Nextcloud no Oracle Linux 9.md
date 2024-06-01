## Instalação e configuração do Nextcloud no Oracle Linux 9

*19 de julho de 2023, Thiago S. Ferreira*

A maioria dos comandos aqui precisam ser executados como usuário `root`, então, para evitar ter que digitar o `sudo` sempre, faça login no shell como `root`:

```
su - root
```

 1. **Habilitar o repositório EPEL:**

    ```
    dnf install -y yum-utils
    dnf config-manager --set-enabled ol9_codeready_builder
    dnf install -y epel-release
    dnf clean all
    dnf update -y
    ```
 2. **Instalar pacotes necessários:**

    ```
    dnf install -y unzip curl wget bash-completion policycoreutils-python-utils mlocate bzip2
    dnf update -y
    ```
 3. **Instalação e configuração do Apache:**

    Instale o Apache com este comando:

    ```
    dnf install -y httpd
    ```

    Crie o arquivo `/etc/httpd/conf.d/nextcloud.conf`, com o seguinte conteúdo:

    ```
    <VirtualHost *:80>
      DocumentRoot /var/www/html/nextcloud/
      ServerName  your.server.com
    
      <Directory /var/www/html/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    
        <IfModule mod_dav.c>
          Dav off
        </IfModule>
    
      </Directory>
    </VirtualHost>
    ```

    Substitua "80" e "your.server.com" pela porta e domínio, respectivamente.

    Habilite, então, o serviço do Apache:

    ```
    systemctl enable httpd.service
    systemctl start httpd.service
    ```
    1. **PHP**

       Adicione o repositório Remi:

       ```
       dnf install -y http://rpms.remirepo.net/enterprise/remi-release-9.rpm
       dnf module reset php
       dnf module install php:remi-8.2
       dnf update -y
       ```

       Instale os pacotes:

       ```
       dnf install -y php php-gd php-mbstring php-intl php-pecl-apcu \
       php-mysqlnd php-opcache php-json php-zip php-redis php-imagick \
       php-opcache php-ldap php-process php-pear php-devel php-gmp \
       php-bcmath
       ```

       Ajuste o limite de memória do PHP para 512M:

       ```
       	
       ```

       Configurar módulo PHP APCU:

       ```
       echo "extension=apcu.so" > /etc/php.d/20-apcu.ini
       echo "apc.enable_cli=1" >> /etc/php.d/20-apcu.ini
       ```

       Ative/reinicie os serviços do PHP e Apache:

       ```
       systemctl enable --now php-fpm.service
       systemctl restart httpd
       ```
 4. **Redis**

    ```
    dnf install -y redis
    systemctl enable redis.service
    systemctl start redis.service
    ```
 5. **Banco de dados MariaDB**

    Instalar o MariaDB:

    ```
    dnf install -y mariadb-server
    ```

    Habilitar o serviço do MariaDB:

    ```
    systemctl enable --now mariadb.service
    ```

    Executar o script secure_installation:

    ```
    mysql_secure_installation
    ```

    Editar o arquivo de configuração, inserindo o seguinte conteúdo no arquivo `/etc/my.cnf.d/mariadb-server.cnf`:

    ```
    [server]
    skip_name_resolve = 1
    innodb_buffer_pool_size = 128M
    innodb_buffer_pool_instances = 1
    innodb_flush_log_at_trx_commit = 2
    innodb_log_buffer_size = 32M
    innodb_max_dirty_pages_pct = 90
    query_cache_type = 1
    query_cache_limit = 2M
    query_cache_min_res_unit = 2k
    query_cache_size = 64M
    tmp_table_size= 64M
    max_heap_table_size= 64M
    slow_query_log = 1
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 1
    
    [client-server]
    !includedir /etc/mysql/conf.d/
    !includedir /etc/mysql/mariadb.conf.d/
    
    [client]
    default-character-set = utf8mb4
    
    [mysqld]
    character_set_server = utf8mb4
    collation_server = utf8mb4_general_ci
    transaction_isolation = READ-COMMITTED
    binlog_format = ROW
    innodb_large_prefix=on
    innodb_file_format=barracuda
    innodb_file_per_table=1
    ```

    Crie o banco de dados e usuário do Nextcloud:

    ```
    mysql -uroot -p
    ```

    No prompt do Mysql, insira estes comandos:

    ```
    CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'Senha';
    CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    GRANT ALL PRIVILEGES on nextcloud.* to 'nextcloud'@'localhost';
    FLUSH privileges;
    quit;
    ```

    (substitua 'Senha' por uma senha definida por você).
 6. **Baixar o Nextcloud e mover para o local apropriado**

    Baixar o pacote, extrair e mover para o local apropriado:

    ```
    wget https://download.nextcloud.com/server/releases/latest.zip
    unzip latest.zip
    mv nextcloud/ /var/www/html/
    ```

    Criar a pasta de dados no local desejado e alterar propriedade e permissão da pasta para pertencer ao usuário `apache`. Neste caso, iremos criar em /Dados/nextcloud/data (neste caso temos uma partição à parte montada em /data):

    ```
    mkdir -p /Dados/nextcloud/data
    chown -R apache:apache /Dados/nextcloud
    chmod 0750 /Dados/nextcloud/data
    ```

    Alterar proprietário e permissões da pasta nextcloud:

    ```
    chown -R apache:apache /var/www/html/nextcloud
    ```

    Reiniciar o Apache:

    ```
    systemctl restart httpd.service
    ```

    Permitir o Apache no firewall:

    ```
    firewall-cmd --zone=public --add-service=http --permanent
    firewall-cmd --zone=public --add-service=https --permanent
    firewall-cmd --reload
    ```
 7. **Configuração do SELinux**

    Execute estes comandos:

    ```
    semanage fcontext -a -t httpd_sys_rw_content_t '/Dados/nextcloud/data(/.*)?'
    semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/config(/.*)?'
    semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/apps(/.*)?'
    semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/.htaccess'
    semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/.user.ini'
    semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'
    
    restorecon -Rv '/var/www/html/nextcloud/'
    restorecon -Rv '/Dados/nextcloud/'
    
    setsebool -P  httpd_unified  on
    setsebool -P httpd_can_connect_ldap on
    setsebool -P httpd_can_network_connect on
    setsebool -P httpd_can_sendmail on
    setsebool -P httpd_use_cifs on
    setsebool -P httpd_use_fusefs on
    setsebool -P httpd_use_gpg on
    ```

    Depois de instalado desative o acesso de gravação a todo o diretório da web:

    ```
    setsebool -P  httpd_unified  off
    ```

    Ative novamente só quando precisar fazer atualização via interface web.
 8. **Acessar a interface e prosseguir com a instalação**

    No navegador insira o endereço IP do servidor nextcloud. Preencha as informações e clique em "Instalar".

 9. **Limite de memória do PHP**
     O Nextcloud recomenda um limite de, pelo menos, 512M para o PHP. Edite o arquivo `/etc/php.ini` e ajuste o valor da variável `memory_limit` para que 512M. Caso seu sistema tenha bastante memória RAM disponível, você pode utilizar um valor maior.
 
 11. **Memory caching**

    Configurar PHP Opcache:

    ```
    sed -i "s/.*opcache.interned_strings_buffer=.*/opcache.interned_strings_buffer=10/g" /etc/php.d/10-opcache.ini
    ```

    Adicione ao final do arquivo config.php do Nextcloud este conteúdo (antes do `);` no final):

    ```
      'memcache.local' => '\OC\Memcache\APCu',
      'memcache.distributed' => '\OC\Memcache\Redis',
      'memcache.locking' => '\OC\Memcache\Redis',
      'redis' => [
           'host' => 'localhost',
           'port' => 6379,
      ],
    ```
11. **Default phone region**

    Adicione ao arquivo config.php do Nextcloud esta linha:

    ```
      'default_phone_region' => 'BR',
    ```
12. **Tarefas em segundo plano**  
    Execute o comando:

    ```
    crontab -u apache -e
    ```

    Este comando abrirá o arquivo Cron do Apache com o editor de texto Vi. Insira esta linha

    ```
    */5  *  *  *  * php -f /var/www/nextcloud/cron.php
    ```

    Acesse as configurações do Nextcloud > Administração > Configurações básicas > Tarefas em segundo plano e selecione a opção "Cron".
