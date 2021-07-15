# wb8-django-elk

## Instalación de paquetes y config de bdd
```
apt-get install -y mariadb-server libmariadbclient-dev default-libmysqlclient-dev build-essential python-dev
mysql_secure_installation
mysql --execute "
	create database django;
	grant all privileges on django.* to 'django_user'@'localhost' identified by 'vahKei4ovah0ieLasohaceiTahxeelae';
	flush privileges;"
```
> Pregunta 1 : Instala mariadb y crea una bbdd para django con su usuario correspondiente. A continuación edita el archivo djangotest/settings.py y configura la base de datos adecuadamente usando el driver de mysql.
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django',
        'USER': 'django_user',
        'PASSWORD': 'vahKei4ovah0ieLasohaceiTahxeelae',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}
pip3 install wheel setuptools
pip3 install mysqlclient
```
Creamos el superuser root:
```
python3 manage.py createsuperuser
```

Arrancamos en http://78.47.148.211:8000/

> Pregunta 2 : Crea un subdominio en cloudflare. Securiza el entorno configurando SSL, instalando un nginx que haga proxy pass a el puerto 8000 de django. Una vez funcione todo correctamente, puedes probar el login por ejemplo.
```
apt install nginx python-certbot-nginx
```
Creamos el dns en cloudflare y el fichero ***/etc/nginx/sites-enabled/django.devops-alumnos.com***
```
	server {
		listen 80;
		server_name django.devops-alumno08.com;
	
		location / {
			proxy_pass http://127.0.0.1:8000;
		}
	}
```
Hay que modificar ***djangotest/settings.py*** y añadir 'localhost' a la lista de HOST_ALLOWED o poner directamente '*'.
Creamos el ssl con `certbot --nginx` y nos quedaría así:
```
	server {
		server_name django.devops-alumno08.com;
	
		location / {
			proxy_pass http://127.0.0.1:8000;
		}
	
	    listen 443 ssl; # managed by Certbot
	    ssl_certificate /etc/letsencrypt/live/django.devops-alumno08.com/fullchain.pem; # managed by Certbot
	    ssl_certificate_key /etc/letsencrypt/live/django.devops-alumno08.com/privkey.pem; # managed by Certbot
	    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
	    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
	
	}
	server {
	    if ($host = django.devops-alumno08.com) {
	        return 301 https://$host$request_uri;
	    } # managed by Certbot
	
	
		listen 80;
		server_name django.devops-alumno08.com;
	    return 404; # managed by Certbot
	
	
	}
```
> Pregunta 3 : Busca como añadir una capa de seguridad extra a nuestra instalación cambiado la ruta de /admin por defecto a opensesame
Modificamos el fichero urls.py y cambiamos admin por opensesame:
```
	urlpatterns = [
	    path('opensesame/', admin.site.urls),
	]
```
> Pregunta 4 : Cambia los permisos de todo el proyecto a www-data. A continuación con sudo encuentra una forma de ejectutar runserver como www-data y comprueba que todo funciona.
```
chown -R www-data: djangotest
su - www-data -s /bin/bash -c "python3 /var/django/djangotest/manage.py runserver 0.0.0.0:8000"
```
> Pregunta 5 : Con que comandos podremos ver si nos falta aplicar alguna migración?
	
 `python3 manage.py showmigrations`
	
> Pregunta 6 : Usando las configuraciones de nginx y uwsgi que tenéis a continuación, acaba la setup. Cuando visites la web puede que fallen los css, es normal, continúa a la pregunta 7.
He usado esta config de nginx:
```
	upstream wsgicluster {
	    server unix:///var/run/uwsgi/django.sock;
	}
	
	server {
	        server_name django.devops-alumno08.com;
	        charset utf-8;
	        # max upload size
	        client_max_body_size 15M;
	
	        location / {
	                include uwsgi_params;
	                uwsgi_pass wsgicluster;
	                #proxy_pass http://127.0.0.1:8000;
	        }
	
	        error_log /var/log/nginx/django.devops-alumno08.com_error.log;
	        access_log /var/log/nginx/django.devops-alumno08.com_access.log;
	
	    listen 443 ssl; # managed by Certbot
	    ssl_certificate /etc/letsencrypt/live/django.devops-alumno08.com/fullchain.pem; # managed by Certbot
	    ssl_certificate_key /etc/letsencrypt/live/django.devops-alumno08.com/privkey.pem; # managed by Certbot
	    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
	    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
	
	}
	server {
	    if ($host = django.devops-alumno08.com) {
	        return 301 https://$host$request_uri;
	    } # managed by Certbot
	
	
	        listen 80;
	        server_name django.devops-alumno08.com;
	    return 404; # managed by Certbot
	
	
	}
```
Y esta configuración de /etc/uwsgi/apps-enabled/django.ini:
```
	[uwsgi]
	
	# Django-related settings
	# the base directory (full path)
	chdir           = /var/django/djangotest
	# Django's wsgi file
	module          = djangotest.wsgi
	# the virtualenv (full path)
	pythonpath	= /var/django/djangotest
	
	# process-related settings
	# master
	master          = true
	# maximum number of worker processes
	processes       = 4
	# the socket (use the full path to be safe
	socket          =  /var/run/uwsgi/django.sock
	# clear environment on exit
	vacuum          = true
	plugin		= python3
```
> Pregunta 7 : Al entregar django desde uwsgi en modo producción no encuentra los archivos estáticos por defecto. Esto es una feature de django. Busca como compilar estos archivos en una carpeta (comando de django) y como configurar adecuandamente nginx para entregar estos archivos.

Hay que cambiar en settings.py la variable DEBUG a false y añadir la ruta de los statics, en mi caso he puesto esta:
`STATIC_ROOT = "/var/www/django.devops-alumno08.com/static/"`
Crear el directorio y darle permisos después de crear los estáticos:
`mkdir -p /var/www/django.devops-alumno08.com/static`
Crear los estáticos:
```
	python3 manage.py collectstatic
	chown -R www-data: /var/www/django.devops-alumno08.com/static
```
Añadir el directorio que vamos usar en nginx:
```
	location /static {
	   root /var/www/django.devops-alumno08.com;
	}
```
Y reinciar los servicios
```
	service uwsgi restart
	service nginx restart
```
> Pregunta 8 : Crea un nuevo subdominio y configura nginx para hacer proxypass a kibana. Cuando funcione instala SSL. Puedes usar esta config de ejemplo:

Para instalar java:
https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-18-04#installing-the-oracle-jdk

Para instalar el resto:
https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-18-04-es

> Pregunta 9 : Securiza el entorno poniendo un htpassword delante.

A continuación instala filebeat. Activa los módulos de nginx, auditd y system.
```
	filebeat setup -e
	for i in auditd nginx system ; do filebeat modules enable $i ; done
```
	
Comprueba que recibimos logs en kibana.

