# Práctica 2.2: host virtual taskflow.local en Apache

Módulo: Despliegue de aplicaciones web (2º DAW)

Alumno: Javier Doñoro Garcia

Fecha: 18 / 11 / 2026

## Introducción

En esta práctica he configurado Apache en Ubuntu Server para que sirva una página de TaskFlow en http://taskflow.local, con sus propios logs. Además he creado un sitio por defecto para que localhost no muestre TaskFlow, y así los dos sitios quedan separados.

## Teoría

### Qué es un VirtualHost

Un VirtualHost es un bloque de configuración de Apache que permite tener varios sitios web en el mismo servidor, con la misma IP y el mismo puerto. Cada bloque tiene su nombre de dominio (ServerName), su carpeta con los ficheros (DocumentRoot) y sus logs (ErrorLog y CustomLog).

### Cómo elige Apache el sitio

Cuando llega una petición, el navegador o curl mandan una cabecera Host con el nombre que se ha escrito, por ejemplo Host: taskflow.local. Apache mira los VirtualHost que escuchan en ese puerto y compara esa cabecera con el ServerName y el ServerAlias de cada uno. Si coincide con alguno, sirve ese sitio.

Si no coincide con ninguno, Apache usa el primer VirtualHost que ha cargado, que es el primero por orden alfabético en sites-enabled. Por eso he llamado al sitio por defecto 00-catchall.conf, para que se cargue antes que taskflow.conf y recoja todo lo que no sea taskflow.local, incluido localhost.

### Cómo añadiría otro host

Para añadir otro sitio, por ejemplo api.taskflow.local, haría lo mismo que en esta práctica: crear su carpeta, crear un fichero api-taskflow.conf en sites-available con su ServerName, su DocumentRoot y sus logs, activarlo con a2ensite, comprobar con apache2ctl configtest, recargar Apache y añadir el nombre en /etc/hosts.

## Paso 0. Instalar Apache

Apache no estaba instalado en la máquina, así que lo primero fue instalarlo y comprobar que estaba en marcha:

```bash
sudo apt update
sudo apt install -y apache2
systemctl status apache2 --no-pager
```

## Paso 1. Crear la carpeta del sitio

Creé la carpeta /var/www/taskflow con un index.html sencillo y le cambié el propietario a www-data, que es el usuario con el que trabaja Apache:

```bash
sudo mkdir -p /var/www/taskflow
echo "<h1>TaskFlow</h1>" | sudo tee /var/www/taskflow/index.html
sudo chown -R www-data:www-data /var/www/taskflow
ls -l /var/www/taskflow
```

Captura 1: ls -l /var/www/taskflow, donde se ve el index.html con propietario www-data.

## Paso 2. Crear taskflow.conf

Creé el fichero de configuración del sitio directamente desde la terminal:

```bash
sudo tee /etc/apache2/sites-available/taskflow.conf <<'EOF'
<VirtualHost *:80>
    ServerName taskflow.local
    ServerAlias www.taskflow.local
    DocumentRoot /var/www/taskflow

    <Directory /var/www/taskflow>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
        FallbackResource /index.html
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/taskflow_error.log
    CustomLog ${APACHE_LOG_DIR}/taskflow_access.log combined
</VirtualHost>
EOF
cat /etc/apache2/sites-available/taskflow.conf
```

Lo más importante es el ServerName, que es el nombre al que responde, y el DocumentRoot, que es la carpeta que sirve. Options -Indexes evita que se liste el contenido de la carpeta, y FallbackResource hace que cualquier ruta que no exista devuelva el index.html. Los logs van a ficheros propios dentro de /var/log/apache2.

Captura 2: cat de taskflow.conf.

## Paso 3. Crear el sitio por defecto

Creé una página distinta para el sitio por defecto y su fichero 00-catchall.conf:

```bash
sudo mkdir -p /var/www/default
echo "<h1>Servidor por defecto</h1>" | sudo tee /var/www/default/index.html
sudo tee /etc/apache2/sites-available/00-catchall.conf <<'EOF'
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/default

    <Directory /var/www/default>
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/default_error.log
    CustomLog ${APACHE_LOG_DIR}/default_access.log combined
</VirtualHost>
EOF
cat /etc/apache2/sites-available/00-catchall.conf
```

Captura 3: cat de 00-catchall.conf.

## Paso 4. Activar los sitios y recargar Apache

Activé los dos sitios nuevos, desactivé el 000-default.conf que viene con Apache y comprobé la sintaxis antes de recargar:

```bash
sudo a2ensite taskflow.conf
sudo a2ensite 00-catchall.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
```

Captura 4: apache2ctl configtest con el resultado Syntax OK.

También sale el aviso AH00558 de que no puede determinar el nombre del servidor (FQDN). Es solo un aviso y no afecta al funcionamiento de los sitios.

Después recargué Apache y comprobé qué sitios quedaban activos:

```bash
sudo systemctl reload apache2
ls -l /etc/apache2/sites-enabled/
```

Captura 5: ls -l de sites-enabled, donde solo aparecen 00-catchall.conf y taskflow.conf.

```bash
sudo apache2ctl -S
```

Captura 6: apache2ctl -S. Se ve que el servidor por defecto es localhost (00-catchall.conf) y que taskflow.local está cargado desde taskflow.conf con el alias www.taskflow.local.

## Paso 5. Añadir taskflow.local a /etc/hosts

Como taskflow.local no es un dominio real, lo añadí a /etc/hosts para que apunte a la propia máquina:

```bash
echo "127.0.0.1   taskflow.local www.taskflow.local" | sudo tee -a /etc/hosts
cat /etc/hosts
```

Captura 7: cat /etc/hosts con la línea de taskflow.local.

## Paso 6. Comprobar con curl

```bash
curl -I http://taskflow.local
curl -s http://taskflow.local
curl -s http://localhost
```

Con curl -I sale HTTP/1.1 200 OK y el servidor Apache/2.4.58 (Ubuntu). Al pedir http://taskflow.local devuelve el h1 de TaskFlow, y al pedir http://localhost devuelve el de Servidor por defecto. Esto demuestra que cada nombre va a su sitio y que localhost no muestra TaskFlow.

Captura 8: las tres peticiones con curl.

## Paso 7. Comprobar los logs separados

```bash
ls -l /var/log/apache2/
sudo tail /var/log/apache2/taskflow_access.log
sudo tail /var/log/apache2/default_access.log
```

En la carpeta de logs aparecen los ficheros de taskflow y los de default. En taskflow_access.log salen las peticiones HEAD y GET que hice a taskflow.local desde 127.0.0.1, y en default_access.log sale el GET a localhost desde ::1. Cada sitio guarda sus peticiones en su propio fichero.

Captura 9: ls de /var/log/apache2 y los tail de los dos logs.

## Conclusión

Con un solo Apache en el puerto 80 tengo dos sitios distintos: taskflow.local, que sirve la página de TaskFlow, y un sitio por defecto que responde a localhost y a cualquier nombre que no esté configurado. Apache los distingue por la cabecera Host y cada uno escribe en sus propios logs. Para añadir más sitios bastaría con repetir los mismos pasos con otro fichero .conf.
