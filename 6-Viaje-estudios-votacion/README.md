
## Crear Dockerfile (ya incluido en app/Dockerfile)
FROM python:3.14.0-alpine3.22
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "wsgi:app"]

## Construir imagen de la app
cd app
docker build -t viaje-estudios .

## Crear red Docker compartida
docker network create webnet

## Ejecutar la app Flask/Gunicorn
docker run -d --name fiesta_app --network webnet viaje-estudios
docker logs fiesta_app  # Verifica que Gunicorn escucha en 0.0.0.0:8000

## Configurar y ejecutar Nginx como reverse proxy

Archivo nginx/default.conf:

server {
    listen 80;
    server_name app.localhost;

    location / {
        proxy_pass http://fiesta_app:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Bonus: cabeceras de seguridad
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options SAMEORIGIN;
        add_header Referrer-Policy no-referrer-when-downgrade;
    }

    location /health {
        return 200 '{"status":"ok"}';
        add_header Content-Type application/json;
    }
}


Ejecutar Nginx en PowerShell:

docker run -d --name nginx_proxy --network webnet -p 8080:80 -v ${PWD.Path}/nginx:/etc/nginx/conf.d:ro nginx:1.27-alpine
docker logs nginx_proxy

## Probar la aplicación

Página principal: http://localhost:8080/

Health check: http://localhost:8080/health
 → {"status":"ok"}

Explicación breve

Flask no se expone directamente porque su servidor integrado no está optimizado para producción y no maneja bien múltiples conexiones.
Nginx actúa como reverse proxy, recibiendo todas las solicitudes de los clientes, gestionando cabeceras, seguridad y tráfico, y reenviándolas a Gunicorn, que ejecuta Flask de manera eficiente.
Esto mejora rendimiento, escalabilidad y seguridad de la aplicación.