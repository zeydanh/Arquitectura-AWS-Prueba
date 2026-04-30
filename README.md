🧱 FASE 1: FIREWALLS (Grupos de Seguridad)
Debes crear tres "cajas fuertes". Ve a EC2 > Red y seguridad > Grupos de seguridad y haz clic en Crear grupo de seguridad.

1. Grupo del Balanceador (Nombre: ALB-SG)


Descripción: Permitir tráfico HTTP mundial.

VPC: Deja la predeterminada.

Reglas de Entrada: * Tipo: HTTP | Puerto: 80 | Origen: Anywhere-IPv4 (0.0.0.0/0).

Haz clic en Crear.

2. Grupo de las Instancias (Nombre: EC2-Web-SG)


Descripción: Proteger servidores. Solo el ALB puede entrar.

Reglas de Entrada:

Regla 1 - Tipo: SSH | Puerto: 22 | Origen: Anywhere-IPv4 (0.0.0.0/0) (o tu IP para seguridad).

Regla 2 - Tipo: TCP Personalizado | Puerto: 8000 | Origen: Personalizado -> Haz clic en el cuadro y selecciona ALB-SG (el grupo que creaste arriba).

Haz clic en Crear.

3. Grupo de Base de Datos (Nombre: RDS-SG)


Descripción: Proteger base de datos. Solo las EC2 entran.

Reglas de Entrada:

Tipo: MySQL/Aurora | Puerto: 3306 | Origen: Personalizado -> Haz clic y selecciona EC2-Web-SG.

Haz clic en Crear.

🎯 FASE 2: TARGET GROUP (El Directorio)
Ve a EC2 > Balanceo de carga > Grupos de destino (Target Groups).

Choose a target type: Instancias.

Target group name: tg-api-fastapi.

Protocol: HTTP | Port: 8000.

VPC: Deja la predeterminada.

Health check path: /health. (En configuración avanzada, confirma Success code: 200).

Haz clic en Siguiente.

🚨 ¡PASO CRÍTICO, TRAMPA DE EXAMEN! 🚨 En la pantalla "Registrar destinos", verás una lista de máquinas. NO SELECCIONES NINGUNA CASILLA. Asegúrate de que abajo diga "0 pendientes". El Auto Scaling meterá las máquinas ahí automáticamente.

Haz clic en Crear grupo de destino.

⚖️ FASE 3: APPLICATION LOAD BALANCER (La Puerta)
Ve a EC2 > Balanceadores de carga > Crear balanceador de carga.

Elige Application Load Balancer y haz clic en Crear.

Load balancer name: alb-api-test.

Scheme: Internet-facing | IP address type: IPv4.

Network mapping (Mapeo de red): Selecciona tu VPC y MÍNIMO DOS zonas (Ej: us-east-1a y us-east-1b).

Security groups: Elimina el grupo default y selecciona tu ALB-SG.

Listeners and routing (Agentes de escucha): Protocol HTTP, Port 80. En "Default action (Forward to)", selecciona tg-api-fastapi.

Ignora servicios adicionales y haz clic en Crear balanceador de carga.

🗄️ FASE 4: BASE DE DATOS (Amazon RDS)
Abre otra pestaña y ve a RDS > Bases de datos > Crear base de datos.

Método de creación: Creación estándar.

Opciones de motor: MySQL.

Plantillas: Capa gratuita (Free tier).

Configuración:

Nombre de la base de datos: db-api-test

Nombre de usuario principal: admin.

Contraseña principal: Admin1234!.

Conectividad:

VPC: La misma de tus EC2.

Acceso público: NO.

Grupo de seguridad de VPC existente: Elimina el default y selecciona RDS-SG.

Despliega la sección de abajo y dale a Crear base de datos. (Tardará unos minutos. Cuando termine, entra y copia el Punto de enlace (Endpoint)).

🧬 FASE 5: LA PLANTILLA (Launch Template)
Vuelve a EC2. Ve a Plantillas de lanzamiento > Crear plantilla de lanzamiento.

Nombre de la plantilla: plantilla-api-fastapi.

Marca la casilla: "Proporcionar orientación para EC2 Auto Scaling".

AMI (SO): Busca y selecciona Amazon Linux 2023.

Tipo de instancia: t3.micro.

Par de claves: Elige tus claves (Ej: mis-claves-aws).

Configuración de red: En Grupo de seguridad, selecciona EC2-Web-SG. NO ELIJAS SUBRED AQUÍ.

🚨 ¡EL CÓDIGO MAESTRO! Ve a Detalles avanzados > Datos de usuario (User Data). Esto asegura que las nuevas máquinas no se caigan por falta del driver pymysql. Pega exactamente esto:

Bash
#!/bin/bash
sudo dnf update -y
sudo dnf install git python3 pip mariadb105 -y
pip install fastapi "uvicorn[standard]" gunicorn pymysql

# Clonar el proyecto e iniciar la aplicación
git clone https://github.com/Elmostunt/LoadBalancerApiTest.git
cd LoadBalancerApiTest

# Iniciar la API en segundo plano
gunicorn -w 3 -k uvicorn.workers.UvicornWorker api:app -b 0.0.0.0:8000 &
Haz clic en Crear plantilla.

⚙️ FASE 6: AUTO SCALING GROUP (La Fábrica)
Ve a EC2 > Grupos de Auto Scaling > Crear grupo de Auto Scaling.

Nombre: asg-api-test. Selecciona tu plantilla-api-fastapi. Siguiente.

Red: Selecciona tu VPC y MÍNIMO 2 SUBREDES (us-east-1a, us-east-1b). Siguiente.

Balanceo de carga: Selecciona "Adjuntar a un equilibrador existente" -> Elige tu Target Group tg-api-fastapi.

🚨 Comprobaciones de estado (Health Checks): Marca Activar ELB. Siguiente.

Tamaño del grupo:

Capacidad deseada: 2.

Mínima: 2.

Máxima: 4.

Políticas de escalado: Selecciona Política de escalado de seguimiento de destino. Métrica: "Utilización media de la CPU". Valor objetivo: 60.

Siguiente hasta el final y haz clic en Crear grupo de Auto Scaling.

🛠️ FASE 7: CONSTRUIR LA TABLA SQL DESDE EC2
Mientras AWS crea tus 2 máquinas, vamos a crear las tablas de tu base de datos.
Ve a EC2 > Instancias. Copia la IP pública de cualquiera de tus 2 nuevas máquinas y entra por SSH desde tu consola (PowerShell/Terminal):

Bash
ssh -i "tu-llave.pem" ec2-user@<IP_PUBLICA_DE_LA_EC2>
Una vez dentro (texto verde [ec2-user@ip-...]), ejecuta:

Descarga el certificado de seguridad:

Bash
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
Conéctate a la BD (Reemplaza <TU_ENDPOINT_RDS> por el que copiaste en Fase 4):

Bash
mysql -h <TU_ENDPOINT_RDS> -P 3306 -u admin -p --ssl-ca=global-bundle.pem --ssl-verify-server-cert
Ingresa tu clave Admin1234! y presiona Enter.

Ejecuta el código SQL:

SQL
CREATE DATABASE api_db;
USE api_db;
CREATE TABLE usuarios ( id INT AUTO_INCREMENT PRIMARY KEY, nombre VARCHAR(100), email VARCHAR(100), fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP );
exit;
🧪 FASE 8: VALIDACIÓN INICIAL
Ve a EC2 > Balanceadores de Carga y copia el DNS de tu alb-api-test.

En el navegador ingresa: http://<DNS_DEL_ALB>/health. Debe responder {"status": "ok"}.

Ingresa a http://<DNS_DEL_ALB>/docs. Usa el método POST para guardar un usuario. Si te responde un JSON verde, ¡ESTÁS CONECTADO A RDS CORRECTAMENTE!

🔥 FASE 9: LA PRUEBA DE JMETER (Rompiendo el sistema)
Para demostrar el Auto Scaling a tu profesor:

Abre Apache JMeter.

Configura el Thread Group:

Number of Threads (users): 500

Ramp-up period: 10

Loop Count: Infinite (o 100)

Configura el HTTP Request:

Protocol: http

Server Name or IP: PEGA AQUÍ TU DNS DEL ALB (¡SIN http:// NI / AL FINAL!)

Port: 80

Path: /health (Para no tener error de Method Not Allowed con /hello).

Dale Play. Ve a AWS > Auto Scaling Groups > Actividad. En unos minutos mostrará "Lanzando nueva instancia EC2".

