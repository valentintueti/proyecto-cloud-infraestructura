# Despliegue en AWS — Paso a Paso

## 1. Red: VPC y subredes

### 1.1 Crear la VPC (wizard "VPC and more")

1. VPC > **Create VPC**.
2. Resources to create: **VPC and more**.
3. Name tag auto-generation: `proyecto-cloud`.
4. IPv4 CIDR block: `10.0.0.0/16`.
5. IPv6 CIDR block: **No IPv6 CIDR block**.
6. Tenancy: **Default**.
7. Number of Availability Zones (AZs): **2**.
8. Number of public subnets: **2**.
9. Number of private subnets: **2**.
10. NAT gateways: **None** (Por tema de pricing y créditos, se creó solo cuando se necesitaba acceder a internet desde las instancias privadas).
11. VPC endpoints: **S3 Gateway**.
12. DNS options: **Enable DNS hostnames** y **Enable DNS resolution**.
13. **Create VPC**.

### 1.2 NAT Gateway (uso temporal, no permanente)

Crear solo cuando `MV-db-server` o `MV-ingesta-pc` necesiten salir a internet (instalar paquetes, `git clone`, `docker pull`):

```
VPC > NAT Gateways > Create NAT gateway
  Subnet: una subred pública
  Connectivity type: Public
  Elastic IP allocation: Allocate Elastic IP
```

Después de usarlo:
```
VPC > NAT Gateways > selecciona el NAT > Delete NAT gateway
VPC > Elastic IPs > selecciona la EIP asociada > Release
```

`MV-app-1` y `MV-app-2` no dependen del NAT porque tienen IP pública propia y salen a internet por el Internet Gateway.

---

## 2. Security Groups

Crear 4 Security Groups, en este orden:

### `alb-sg`
1. Name: `alb-sg`. VPC: `proyecto-cloud-vpc`.
2. Inbound rules: Custom TCP, Port range `8081-8085`, Source `0.0.0.0/0` (o `10.0.0.0/16` para mayor restricción).

### `app-sg`
1. Name: `app-sg`. VPC: `proyecto-cloud-vpc`.
2. Inbound rule: Custom TCP, Port range `8081-8085`, Source: `alb-sg`.
3. Inbound rule: SSH, Port `22`, Source: `0.0.0.0/0`.

### `db-sg`
1. Name: `db-sg`. VPC: `proyecto-cloud-vpc`.
2. Inbound rule: Custom TCP, Port `3306`, Source: `app-sg`.
3. Inbound rule: Custom TCP, Port `5432`, Source: `app-sg`.
4. Inbound rule: Custom TCP, Port `27017`, Source: `app-sg`.
5. Inbound rule: SSH, Port `22`, Source: `app-sg`.

### `ingesta-sg`
1. Name: `ingesta-sg`. VPC: `proyecto-cloud-vpc`.
2. Add rule: SSH, Port `22`, Source: `app-sg`.

---

## 3. Key Pair para SSH

1. EC2 > Key Pairs > Create key pair.
2. Name: `proyecto-cloud-key`.
3. Key pair type: RSA. Private key file format: `.pem`.

GUARDAR EL .pem

---

## 4. Lanzamiento de las 4 instancias EC2

### User data (idéntico en las 4 instancias)

```bash
#!/bin/bash
apt-get update -y
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin git
systemctl enable docker
systemctl start docker
usermod -aG docker ubuntu
```


### 4.1 MV-app-1

1. Name: `MV-app-1`.
2. Application and OS Images: Quick Start > **Ubuntu** > seleccionar **Ubuntu Server 24.04 LTS (HVM)**.
3. Instance type: `t3.medium`.
4. Key pair (login): `proyecto-cloud-key`.
5. Network settings:
   - VPC: `proyecto-cloud-vpc`
   - Subnet: una subred **pública** (AZ 1)
   - Auto-assign public IP: **Enable**
   - Select existing security group > `app-sg`
6. Configure storage: **8 GiB**.
7. Advanced details:
   - IAM instance profile: `LabInstanceProfile`
   - User data: pegar el script de arriba
8. Launch instance.

### 4.2 MV-app-2

1. Name: `MV-app-2`.
2. Application and OS Images: Quick Start > **Ubuntu** > seleccionar **Ubuntu Server 24.04 LTS (HVM)**.
3. Instance type: `t3.medium`.
4. Key pair (login): `proyecto-cloud-key`.
5. Network settings > Edit:
   - VPC: `proyecto-cloud-vpc`
   - Subnet: la **otra** subred **pública** (AZ 2)
   - Auto-assign public IP: **Enable**
   - Select existing security group > `app-sg`
6. Configure storage: **8 GiB**.
7. Advanced details:
   - IAM instance profile: `LabInstanceProfile`
   - User data: pega el mismo script
8. Launch instance.

### 4.3 MV-db-server

1. Name: `MV-db-server`.
2. Application and OS Images: Ubuntu Server 24.04 LTS (HVM).
3. Instance type: `t3.medium`.
4. Key pair (login): `proyecto-cloud-key`.
5. Network settings > Edit:
   - VPC: `proyecto-cloud-vpc`
   - Subnet: una subred **privada**
   - Auto-assign public IP: **Disable**
   - Select existing security group > `db-sg`
6. Configure storage: **20 GiB**.
7. Advanced details:
   - IAM instance profile: `LabInstanceProfile`
   - User data: pega el mismo script
8. Launch instance.

### 4.4 MV-ingesta-pc

1. Name: `MV-ingesta-pc`.
2. Application and OS Images: Ubuntu Server 24.04 LTS (HVM).
3. Instance type: `t3.medium`.
4. Key pair (login): `proyecto-cloud-key`.
5. Network settings > Edit:
   - VPC: `proyecto-cloud-vpc`
   - Subnet: una subred **privada** (puede ser la misma que `MV-db-server` u otra AZ privada)
   - Auto-assign public IP: **Disable**
   - Select existing security group > `ingesta-sg`
6. Configure storage: **8 GiB**.
7. Advanced details:
   - IAM instance profile: `LabInstanceProfile`
   - User data: pega el mismo script
8. Launch instance.

---

## 5. Conexión a las instancias

### A MV-app-1 / MV-app-2 (públicas)

1. EC2 > Instances > selecciona la instancia > **Connect**.
2. Pestaña **EC2 Instance Connect** > Connect.

### A MV-db-server / MV-ingesta-pc (privadas)

1. Conéctate primero a `MV-app-1`.
2. Pega la llave privada dentro de esa terminal:
   ```bash
   nano proyecto-cloud-key.pem
   ```
   Pega el contenido completo del archivo `.pem`.
3. Ajusta permisos:
   ```bash
   chmod 400 proyecto-cloud-key.pem
   ```
4. Obtén la IP privada de la instancia destino: EC2 > selecciona la instancia > pestaña Details > Private IPv4 address.
5. Salta a la instancia privada:
   ```bash
   ssh -i proyecto-cloud-key.pem ubuntu@IP_PRIVADA_DESTINO
   ```

---

## 6. Despliegue de las bases de datos (MV-db-server)

1. Conectarse a `MV-db-server`.
2. Clonar el repositorio:
   ```bash
   git clone https://github.com/valentintueti/poyecto-cloud-infraestructura.git
   cd repo-infraestructura/databases
   ```
3. Crear el archivo `.env` necesario :
   ```
   POSTGRES_PASSWORD=<password_postgres>
   MYSQL_PASSWORD=<password_mysql>
   MYSQL_ROOT_PASSWORD=<password_root_mysql>
   ```
4. Levantar los contenedores:
   ```bash
   docker compose up -d --build
   ```

---

## 7. Despliegue de los microservicios (MS1-MS5)

Repetir en **ambas** `MV-app-1` y `MV-app-2`.

1. Conéctate a la instancia.
2. Clona los repositorios:
   ```bash
   git clone https://github.com/valentintueti/proyecto-cloud-infraestructura.git
   git clone https://github.com/valentintueti/proyecto-cloud-ms1.git
   git clone https://github.com/valentintueti/proyecto-cloud-ms2.git
   git clone https://github.com/valentintueti/proyecto-cloud-ms3.git
   git clone https://github.com/valentintueti/proyecto-cloud-ms4.git 
   git clone https://github.com/valentintueti/proyecto-cloud-ms5.git 
   ```
3. Ir a la carpeta de apps del repo de infraestructura:
   ```bash
   cd repo-infraestructura/apps
   ```
4. Crear los archivos `.env` de cada microservicio:
   ```bash
   nano ms1.env
   nano ms2.env
   nano ms3.env
   nano ms4.env
   nano ms5.env
   ```
   Con el siguiente contenido:
   ```ms1.env
   DB_HOST=<ip_privada_MV-db-server>
   DB_PORT=5432
   DB_NAME=ms1_pasajeros
   DB_USER=<usuario_postgres>
   DB_PASSWORD=<password_postgres>
   ```
   ```ms2.env
   MONGO_URI=mongodb://<ip_privada_MV-db-server>:27017
   DB_NAME=ms2_servicios
   ROOT_PATH=/ms2
   ```

   ```ms3.env
   PORT=3000
   DB_HOST=<ip_privada_MV-db-server>
   DB_PORT=3306
   DB_NAME=ms3_viajes
   DB_USER=<usuario_mysql>
   DB_PASSWORD=<password_mysql>
   MS1_BASE_URL=http://ms1-app:8081
   MS2_BASE_URL=http://ms2-app:8000
   ROOT_PATH=/ms3
   ```

   ```ms4.env
   MS1_BASE_URL=http://ms1-app:8081
   MS2_BASE_URL=http://ms2-app:8000
   MS3_BASE_URL=http://ms3-app:3000
   ROOT_PATH=/ms4
   ```

   ```ms5.env
   AWS_REGION=us-east-1
   ATHENA_DATABASE=transporte
   ATHENA_OUTPUT_LOCATION=s3://tu-bucket/athena-results/
   ROOT_PATH=/ms5
   AWS_ACCESS_KEY_ID=<credencial_temporal_learner_lab>
   AWS_SECRET_ACCESS_KEY=<credencial_temporal_learner_lab>
   AWS_SESSION_TOKEN=<credencial_temporal_learner_lab>
   ```
   Las 3 credenciales `AWS` son necesarias porque en Learner Lab el `LabInstanceProfile` de la instancia no siempre es accesible de forma confiable desde dentro de un contenedor Docker (el endpoint de metadata `169.254.169.254` puede no estar alcanzable según la red del contenedor). Se obtienen del panel **AWS Details** de tu sesión de Academy y **expiran cada 4 horas** — hay que actualizarlas y volver a levantar el contenedor cuando caduquen.
5. Construye y levanta:
   ```bash
   docker compose up -d --build
   ```

---

## 8. Application Load Balancer y Target Groups

### 8.1 Target Groups (uno por microservicio)

Repite por cada uno (`ms1-tg` a `ms5-tg`, puertos 8081 a 8085):

1. Target type: **Instances**.
2. Name: `ms1-tg` (o el que corresponda).
3. Protocol: HTTP. Port: `8081` (o el correspondiente).
4. VPC: `proyecto-cloud-vpc`.
5. Health check path: `/health`.
6. Next > selecciona `MV-app-1` y `MV-app-2`, especifica el puerto > Include as pending below > Create target group.

### 8.2 Application Load Balancer

1. EC2 > Load Balancers > Create load balancer > Application Load Balancer.
2. Name: `app-alb`.
3. Scheme: **Internal**.
4. VPC: `proyecto-cloud-vpc`.
5. Subnets: las 2 subredes privadas.
6. Security group: `alb-sg`.
7. Listeners: agrega uno HTTP en el puerto `8081`, forward to `ms1-tg`.
8. Create load balancer.
9. Una vez creado, ve a la pestaña **Listeners and rules** > Add listener, y repite para los puertos `8082` (→`ms2-tg`), `8083` (→`ms3-tg`), `8084` (→`ms4-tg`), `8085` (→`ms5-tg`).

Espera 2-3 minutos y confirma en cada Target Group que el estado pase a **healthy**.

---

## 9. API Gateway (HTTP API + VPC Link v2)

### 9.1 VPC Link

1. API Gateway > VPC links > Create.
2. Version: **VPC Link for HTTP APIs**.
3. Name: `vpc-link-microservicios`.
4. Target: `app-alb`.
5. Subnets: las 2 privadas.
6. Security group: `alb-sg`.
7. Create. Espera a que pase a `AVAILABLE`.

### 9.2 HTTP API

1. API Gateway > APIs > Create API > HTTP API > Build.
2. Name: `api-microservicios`.

### 9.3 Rutas e integraciones (por cada microservicio, ms1 a ms5)

1. Routes > Create. Crea una ruta por cada método que use el frontend (GET, POST, PUT, DELETE, PATCH según corresponda) hacia `/msX/{proxy+}`. No crear ruta `ANY`.
2. En la primera ruta de ese microservicio, ve a Integration > **Create and attach an integration**:
   - Integration type: Private resource.
   - Integration target type: Application Load Balancer/Network Load Balancer.
   - Load balancer: `app-alb`.
   - Listener: el del puerto correspondiente (8081 para ms1, etc.).
   - VPC link: `vpc-link-microservicios`.
3. En las demás rutas del mismo microservicio, ve a Integration > **Attach an existing integration** y selecciona la ya creada.
4. En esa integración (una sola por microservicio), ve a **Parameter Mappings** > Add:
   - Parameter to modify: `path`
   - Modification type: `Overwrite`
   - Value: `/$request.path.proxy`
5. Repite el punto 1-4 para ms2, ms3, ms4, ms5.

### 9.4 CORS

1. API Gateway > tu API > CORS.
2. Access-Control-Allow-Origin: `*` (o el dominio de Amplify).
3. Access-Control-Allow-Methods: `GET, POST, PUT, DELETE, PATCH, OPTIONS`.
4. Access-Control-Allow-Headers: `Content-Type, Authorization`.
5. Save.

### 9.5 Deploy

El stage `$default` se autodesplega. Copia la Invoke URL (`https://xxxxx.execute-api.us-east-1.amazonaws.com`).

---

## 10. Bucket S3 e ingesta de datos

### 10.1 Bucket

1. S3 > Create bucket.
2. Name: `proyecto-cloud-ingesta-<identificador-unico>`.
3. Block all public access: activado.
4. Create bucket.

### 10.2 Estructura del repo de ingesta

La ingesta está dentro del mismo repositorio de MS5, en la carpeta `ingesta/`, junto a la API que consulta Athena (`api/`). Son 3 microservicios de ingesta independientes, uno por base de datos origen, sin código compartido entre ellos

Resultado final en el bucket (7 prefijos, sin subcarpetas de fecha):
```
s3://tu-bucket/
├── pasajeros/data.csv       (MS1)
├── tarjetas/data.csv        (MS1)
├── rutas/data.json          (MS2)
├── paraderos/data.json      (MS2)
├── servicios/data.json      (MS2)
├── viajes/data.csv          (MS3)
└── conexiones/data.csv      (MS3)
```

### 10.3 Despliegue en MV-ingesta-pc

1. Conéctate a `MV-ingesta-pc` (igual que se conecta a MV-db-server, con NAT temporal activo, ver sección 1.2 — necesitas salir a internet para `git clone` y `docker pull`).
2. Clona el repositorio de MS5:
   ```bash
   git clone https://github.com/tu-usuario/repo-ms5.git proyecto-cloud-ms5
   cd proyecto-cloud-ms5/ingesta
   ```
3. Crea el `.env` de cada uno de los 3 microservicios de ingesta:
   ```bash
   nano ingesta-ms1/.env
   ```
   ```
   DB_HOST=<ip_privada_MV-db-server>
   DB_PORT=5432
   DB_NAME=ms1_pasajeros
   DB_USER=<usuario_postgres>
   DB_PASSWORD=<password_postgres>
   S3_BUCKET=tu-bucket
   AWS_REGION=us-east-1
   AWS_ACCESS_KEY_ID=<credencial_temporal_learner_lab>
   AWS_SECRET_ACCESS_KEY=<credencial_temporal_learner_lab>
   AWS_SESSION_TOKEN=<credencial_temporal_learner_lab>
   ```
   ```bash
   nano ingesta-ms2/.env
   ```
   ```
   MONGO_URI=mongodb://<ip_privada_MV-db-server>:27017
   DB_NAME=ms2_servicios
   S3_BUCKET=tu-bucket
   AWS_REGION=us-east-1
   AWS_ACCESS_KEY_ID=<credencial_temporal_learner_lab>
   AWS_SECRET_ACCESS_KEY=<credencial_temporal_learner_lab>
   AWS_SESSION_TOKEN=<credencial_temporal_learner_lab>
   ```
   ```bash
   nano ingesta-ms3/.env
   ```
   ```
   DB_HOST=<ip_privada_MV-db-server>
   DB_PORT=3306
   DB_NAME=ms3_viajes
   DB_USER=<usuario_mysql>
   DB_PASSWORD=<password_mysql>
   S3_BUCKET=tu-bucket
   AWS_REGION=us-east-1
   AWS_ACCESS_KEY_ID=<credencial_temporal_learner_lab>
   AWS_SECRET_ACCESS_KEY=<credencial_temporal_learner_lab>
   AWS_SESSION_TOKEN=<credencial_temporal_learner_lab>
   ```
   Las credenciales `AWS_*` salen del panel **AWS Details** de Learner Lab y expiran cada pocas horas — hay que refrescarlas y volver a correr los contenedores cuando caduquen.
4. Levanta los 3 contenedores (corren una vez y terminan, no quedan en segundo plano):
   ```bash
   docker compose up --build
   ```

### 10.4 Ejecución periódica con cron (opcional)

1. Abre el crontab:
   ```bash
   crontab -e
   ```
2. Agrega al final (ejecución cada hora; ajusta la ruta si clonaste en otro lugar):
   ```
   0 * * * * cd /home/ubuntu/proyecto-cloud-ms5/ingesta && docker compose up --build >> /var/log/ingesta.log 2>&1
   ```

   Nota: si usas credenciales temporales de Learner Lab, el cron va a fallar en cuanto expiren — hay que actualizar los 3 `.env` manualmente cada vez que se renueve la sesión de Academy, el cron no puede refrescarlas solo.

---

## 11. AWS Glue: catálogo de datos

### 11.1 Base de datos del catálogo

1. Glue > Data Catalog > Databases > Add database.
2. Name: `transporte` (debe coincidir con `ATHENA_DATABASE` en `ms5.env`).
3. Create database.

### 11.2 Crawler

1. Glue > Crawlers > Create crawler.
2. Name: `crawler-transporte`.
3. Next.
4. Data source: Add a data source > S3 path: `s3://tu-bucket/pasajeros/` > Add S3 data source.
5. Repite "Add a data source" por cada uno de los 7 prefijos:
   - `s3://tu-bucket/pasajeros/`
   - `s3://tu-bucket/tarjetas/`
   - `s3://tu-bucket/rutas/`
   - `s3://tu-bucket/paraderos/`
   - `s3://tu-bucket/servicios/`
   - `s3://tu-bucket/viajes/`
   - `s3://tu-bucket/conexiones/`
6. Next.
7. IAM role: `LabRole`.
8. Next. Target database: `transporte`.
9. Frequency: On demand.
10. Next > Create crawler.
11. Selecciona el crawler > **Run crawler**.
12. Verifica en Glue > Tables que se hayan creado exactamente 7 tablas, una por prefijo, con el mismo nombre (`pasajeros`, `tarjetas`, `rutas`, `paraderos`, `servicios`, `viajes`, `conexiones`).

---

## 12. AWS Athena: vistas y queries

### 12.1 Configuración inicial

1. Athena > Settings > Manage.
2. Location of query result: `s3://tu-bucket/athena-results/` (debe coincidir exactamente con `ATHENA_OUTPUT_LOCATION` en `ms5.env`, sección 7).
3. Save.

### 12.2 Vistas

Corre en el Query editor, con la base `transporte` seleccionada.

```sql
CREATE OR REPLACE VIEW transporte.vista_viajes_completos AS
SELECT
    v.id AS viaje_id,
    COALESCE(
        TRY(date_parse(v.fecha_hora, '%Y-%m-%d %H:%i:%s')),
        TRY(date_parse(v.fecha_hora, '%Y-%m-%dT%H:%i:%s'))
    ) AS fecha_hora,
    r.nombre AS ruta_nombre,
    r.sentido AS ruta_sentido,
    pd.nombre AS paradero_origen
FROM transporte.viajes v
JOIN transporte.servicios s ON v.servicio_id = s.id
JOIN transporte.rutas r ON s.ruta_id = r.id
JOIN transporte.paraderos pd ON v.paradero_origen_id = pd.id;
```

```sql
CREATE OR REPLACE VIEW transporte.vista_conexiones_completas AS
SELECT
    c.paradero_id,
    pd.nombre AS paradero_nombre,
    r_destino.nombre AS ruta_destino,
    r_origen.nombre AS ruta_origen,
    c.fecha_hora
FROM transporte.conexiones c
JOIN transporte.viajes v_destino ON c.viaje_destino_id = v_destino.id
JOIN transporte.servicios s_destino ON v_destino.servicio_id = s_destino.id
JOIN transporte.rutas r_destino ON s_destino.ruta_id = r_destino.id
JOIN transporte.viajes v_origen ON c.viaje_origen_id = v_origen.id
JOIN transporte.servicios s_origen ON v_origen.servicio_id = s_origen.id
JOIN transporte.rutas r_origen ON s_origen.ruta_id = r_origen.id
JOIN transporte.paraderos pd ON c.paradero_id = pd.id;
```



### 12.3 Queries

```sql
-- Consulta 1: demanda por ruta
SELECT ruta_nombre, ruta_sentido, COUNT(*) AS total_viajes
FROM transporte.vista_viajes_completos
GROUP BY ruta_nombre, ruta_sentido
ORDER BY total_viajes DESC;
```

```sql
-- Consulta 2: demanda por paradero
SELECT paradero_origen, COUNT(*) AS total_viajes
FROM transporte.vista_viajes_completos
GROUP BY paradero_origen
ORDER BY total_viajes DESC;
```

```sql
-- Consulta 3: demanda por hora del día, por ruta
SELECT ruta_nombre, hour(fecha_hora) AS hora_del_dia, COUNT(*) AS total_viajes
FROM transporte.vista_viajes_completos
GROUP BY ruta_nombre, hour(fecha_hora)
ORDER BY ruta_nombre, hora_del_dia;
```

```sql
-- Consulta 4: rutas que más reciben trasbordos
SELECT ruta_destino, COUNT(*) AS total_conexiones
FROM transporte.vista_conexiones_completas
GROUP BY ruta_destino
ORDER BY total_conexiones DESC;
```


---

## 13. Frontend en AWS Amplify

1. Amplify > Create new app > Host web app > GitHub.
2. Autoriza la GitHub App (Repository access: All repositories, o selecciona el repo del frontend).
3. Selecciona el repositorio y la rama (`main`).
4. Revisa el `amplify.yml` autodetectado.
5. Advanced settings > Environment variables, agrega:
   ```
   VITE_MS1_BASE_URL=https://tu-invoke-url.execute-api.us-east-1.amazonaws.com/ms1
   VITE_MS2_BASE_URL=https://tu-invoke-url.execute-api.us-east-1.amazonaws.com/ms2
   VITE_MS3_BASE_URL=https://tu-invoke-url.execute-api.us-east-1.amazonaws.com/ms3
   VITE_MS4_BASE_URL=https://tu-invoke-url.execute-api.us-east-1.amazonaws.com/ms4
   VITE_MS5_BASE_URL=https://tu-invoke-url.execute-api.us-east-1.amazonaws.com/ms5
   ```
6. Save and deploy.
