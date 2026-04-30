

# 🚀 FastAPI + AWS Auto Scaling + Load Balancer

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-green?logo=fastapi)
![Status](https://img.shields.io/badge/Status-Running-success)

---

## 📌 Descripción

Este proyecto despliega una API hecha en **FastAPI** sobre AWS utilizando una arquitectura escalable:

* ⚖️ **Application Load Balancer**
* 🔄 **Auto Scaling Group**
* 🗄️ **Amazon RDS (MySQL)**
* 🛡️ **Security Groups**
* 🐳 Backend con **Gunicorn + Uvicorn**

---

## 🧠 Arquitectura

```text
Internet
   ↓
ALB (HTTP :80)
   ↓
EC2 (Auto Scaling Group)
   ↓
FastAPI (Puerto 8000)
   ↓
RDS MySQL (Puerto 3306)
```

---

## ⚙️ Configuración paso a paso

---

# 🧱 1. Security Groups

Crear en:

**EC2 > Security Groups**

### 🔹 ALB-SG

Permite tráfico web público

* HTTP | 80 | 0.0.0.0/0

---

### 🔹 EC2-Web-SG

Protege las instancias

* SSH | 22 | tu IP (recomendado)
* TCP | 8000 | ALB-SG

---

### 🔹 RDS-SG

Protege la base de datos

* MySQL | 3306 | EC2-Web-SG

---

# 🎯 2. Target Group

**EC2 > Target Groups**

* Nombre: `tg-api-fastapi`
* Tipo: Instancias
* Protocolo: HTTP
* Puerto: 8000
* Health Check: `/health`

🚨 No agregar instancias manualmente

---

# ⚖️ 3. Load Balancer

**Application Load Balancer**

* Nombre: `alb-api-test`
* Esquema: Internet-facing
* Subredes: mínimo 2
* Security Group: ALB-SG

### Listener

* HTTP :80 → `tg-api-fastapi`

---

# 🗄️ 4. Base de datos (RDS)

**RDS > Crear DB**

* Motor: MySQL
* DB Name: `db-api-test`
* User: `admin`
* Pass: `Admin1234!`

### Conectividad

* Acceso público: ❌
* SG: `RDS-SG`

📌 Guardar el **endpoint**

---

# 🧬 5. Launch Template

**EC2 > Launch Templates**

* Nombre: `plantilla-api-fastapi`
* AMI: Amazon Linux 2023
* Tipo: t3.micro
* SG: EC2-Web-SG

---

## 📜 User Data

```bash
#!/bin/bash

sudo dnf update -y
sudo dnf install -y git python3 pip mariadb105

pip install fastapi "uvicorn[standard]" gunicorn pymysql

git clone https://github.com/Elmostunt/LoadBalancerApiTest.git
cd LoadBalancerApiTest

gunicorn -w 3 -k uvicorn.workers.UvicornWorker api:app -b 0.0.0.0:8000 &
```

---

# ⚙️ 6. Auto Scaling Group

**EC2 > Auto Scaling Groups**

* Nombre: `asg-api-test`
* Template: `plantilla-api-fastapi`

### Configuración

* Subredes: mínimo 2
* Target Group: `tg-api-fastapi`
* Health Check: ELB

### Capacidad

* Min: 2
* Desired: 2
* Max: 4

### Scaling

* Tipo: Target Tracking
* CPU: 60%

---

# 🛠️ 7. Configurar base de datos

### Conectarse por SSH

```bash
ssh -i "tu-llave.pem" ec2-user@<IP>
```

### Descargar certificado

```bash
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

### Conectar a MySQL

```bash
mysql -h <ENDPOINT> -P 3306 -u admin -p \
--ssl-ca=global-bundle.pem --ssl-verify-server-cert
```

---

### Crear tablas

```sql
CREATE DATABASE api_db;
USE api_db;

CREATE TABLE usuarios (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100),
  correo VARCHAR(100),
  fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

# 🧪 8. Validación

### Health check

```bash
http://<DNS_ALB>/health
```

### Documentación

```bash
http://<DNS_ALB>/docs
```

---

# 🔥 9. Prueba de carga (JMeter)

### Thread Group

* Users: 500
* Ramp-up: 10
* Loop: 100

### HTTP Request

* Host: DNS del ALB
* Puerto: 80
* Path: `/health`

---

## 📈 Resultado esperado

El sistema debería:

* Escalar automáticamente
* Crear nuevas instancias
* Balancear tráfico
* Mantener disponibilidad

---

# ✅ Checklist

* [ ] Security Groups creados
* [ ] Target Group listo
* [ ] ALB funcionando
* [ ] RDS conectado
* [ ] Auto Scaling activo
* [ ] API respondiendo
* [ ] Prueba de carga exitosa

---

# 🧩 Mejoras futuras

* HTTPS con certificado SSL (ACM)
* Dominio personalizado
* CI/CD con GitHub Actions
* Dockerización
* Logging con CloudWatch

---

Si quieres el siguiente nivel 🚀 te puedo hacer:

* diagrama visual tipo draw.io (para que lo subas al README)
* portada con imagen gamer tipo tu proyecto "Nordic Legends"
* o dejarlo como **README ganador de evaluación** con storytelling 👀
