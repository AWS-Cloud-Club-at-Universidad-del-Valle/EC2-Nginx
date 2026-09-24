# Tutorial: Conectarse a una EC2 (Ubuntu), instalar Nginx y acceder desde el navegador

Guía paso a paso para conectarte a una instancia EC2 de AWS con **Ubuntu** por SSH, ajustar los permisos de la clave `.pem`, instalar el servidor web Nginx y comprobar que funciona desde tu navegador.

---

## Requisitos previos

- Una instancia **EC2 con Ubuntu** en ejecución (estado `running`) en la consola de AWS.
- El archivo de clave privada `.pem` que descargaste al crear el par de claves (key pair).
- La **IP pública** o el **DNS público** de la instancia (visible en la consola de EC2).
- Un cliente SSH:
  - **macOS / Linux**: viene incluido en la terminal.
  - **Windows**: usa PowerShell, WSL o [PuTTY](https://www.putty.org/).

> En Ubuntu el usuario por defecto para conectarse es **`ubuntu`**.

---

## Paso 1: Configurar el Security Group

Antes de conectarte, la instancia debe permitir el tráfico entrante. En la consola de AWS, edita el **Security Group** asociado a tu EC2 y agrega estas reglas de entrada (inbound):

| Tipo   | Protocolo | Puerto | Origen                       | Para qué                    |
|--------|-----------|--------|------------------------------|-----------------------------|
| SSH    | TCP       | 22     | Mi IP (recomendado)          | Conectarte por SSH          |
| HTTP   | TCP       | 80     | 0.0.0.0/0 (cualquier lugar)  | Acceder al sitio web        |

> **Buena práctica de seguridad**: para SSH usa "Mi IP" en lugar de `0.0.0.0/0` para no exponer el puerto 22 a todo internet.

### Alternativa: abrir el puerto 80 con AWS CLI

Si prefieres la línea de comandos en vez de la consola web, puedes abrir el puerto 80 con la AWS CLI. Reemplaza los valores entre `<...>` por los tuyos.

1. Localiza tu instancia y su Security Group:

```bash
aws ec2 describe-instances \
  --query "Reservations[].Instances[].{ID:InstanceId,IP:PublicIpAddress,SG:SecurityGroups[].GroupId}" \
  --output table
```

2. Revisa las reglas de entrada actuales del Security Group:

```bash
aws ec2 describe-security-groups \
  --group-ids <SECURITY_GROUP_ID> \
  --query "SecurityGroups[].IpPermissions" --output json
```

3. Agrega la regla de entrada HTTP (puerto 80) abierta a internet:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <SECURITY_GROUP_ID> \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

> Si tienes varios perfiles de AWS CLI o trabajas en otra región, añade `--profile <TU_PERFIL>` y `--region <TU_REGION>` a cada comando.

---

## Paso 2: Dar permisos correctos al archivo .pem

SSH **rechaza** claves privadas con permisos demasiado abiertos. La clave debe ser legible solo por tu usuario.

Abre la terminal, ubícate en la carpeta donde está tu `.pem` y ejecuta:

```bash
chmod 400 "ec2 clave.pem"
```

- `400` significa: solo el propietario puede leer, nadie más puede leer/escribir/ejecutar.
- Las comillas son necesarias porque el nombre del archivo tiene un espacio.

Verifica los permisos:

```bash
ls -l "ec2 clave.pem"
```

Deberías ver algo como:

```
-r--------  1 tuusuario  staff  1704  ec2 clave.pem
```

> Si no ajustas esto, SSH mostrará el error `UNPROTECTED PRIVATE KEY FILE!` y no te dejará conectar.

---

## Paso 3: Conectarse a la EC2 por SSH

Usa el siguiente comando reemplazando `TU_IP_PUBLICA` por la IP pública de tu instancia:

```bash
ssh -i "ec2 clave.pem" ubuntu@TU_IP_PUBLICA
```

Ejemplo:

```bash
ssh -i "ec2 clave.pem" ubuntu@54.210.123.45
```

- `-i` indica el archivo de identidad (tu clave privada).
- `ubuntu` es el usuario por defecto en las AMIs de Ubuntu.

La primera vez verás un mensaje sobre la autenticidad del host:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Escribe `yes` y presiona Enter. Si todo va bien, verás el prompt de la instancia:

```
ubuntu@ip-172-31-xx-xx:~$
```

---

## Paso 4: Instalar Nginx

Ya conectado a la instancia, actualiza los paquetes e instala Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

---

## Paso 5: Iniciar y habilitar Nginx

En Ubuntu, Nginx suele arrancar automáticamente tras instalarse. Aun así, asegúrate de que esté iniciado y habilitado para que se levante solo cuando la instancia se reinicie:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Comprueba que está corriendo:

```bash
sudo systemctl status nginx
```

Busca la línea `active (running)`. Presiona `q` para salir de la vista de estado.

---

## Paso 6: Acceder al servidor web desde el navegador

1. Copia la **IP pública** (o DNS público) de tu instancia desde la consola de EC2.
2. Ábrela en tu navegador:

```
http://TU_IP_PUBLICA
```

Ejemplo:

```
http://54.210.123.45
```

Deberías ver la página de bienvenida de Nginx:

> **Welcome to nginx!**

¡Listo! Tu servidor web ya está funcionando y accesible desde internet.

---

## Paso 7 (opcional): Publicar tu propia página

En Ubuntu, la raíz web por defecto de Nginx es `/var/www/html`. Puedes reemplazar la página por defecto con tu propio contenido HTML:

```bash
echo "<h1>Hola desde mi EC2 con Nginx</h1>" | sudo tee /var/www/html/index.html
```

Refresca el navegador para ver tu página.

---

## Nota sobre el firewall (UFW)

Si tienes activo el firewall `ufw` en Ubuntu, permite el tráfico web:

```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw status
```

> Por defecto, en una EC2 recién creada `ufw` suele estar **inactivo** y el control de acceso lo maneja el Security Group. Solo necesitas esto si activaste `ufw` manualmente.

---

## Solución de problemas

| Problema | Causa probable | Solución |
|----------|----------------|----------|
| `UNPROTECTED PRIVATE KEY FILE!` | Permisos del `.pem` muy abiertos | `chmod 400 "ec2 clave.pem"` |
| `Permission denied (publickey)` | Usuario o clave incorrectos | Verifica que uses `ubuntu@...` y el `.pem` correcto |
| `Connection timed out` al conectar | Puerto 22 cerrado en el Security Group | Abre el puerto 22 desde "Mi IP" |
| El navegador no carga la página | Puerto 80 cerrado o Nginx detenido | Abre el puerto 80 en el Security Group y verifica `systemctl status nginx` |
| La página tarda o no responde | Usas `https://` | Prueba con `http://` (sin certificado no hay HTTPS) |

---

## Comandos útiles de Nginx

```bash
sudo systemctl restart nginx   # Reiniciar Nginx
sudo systemctl stop nginx      # Detener Nginx
sudo nginx -t                  # Probar la configuración
sudo tail -f /var/log/nginx/access.log   # Ver accesos en tiempo real
sudo tail -f /var/log/nginx/error.log    # Ver errores en tiempo real
```

---

¡Con esto tienes tu servidor web Nginx corriendo en tu EC2 con Ubuntu y accesible desde el navegador!
