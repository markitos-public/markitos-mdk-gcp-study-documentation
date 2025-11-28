
# 📜 004: Cloud SQL

## 📝 Índice

1.  [Descripción](#descripción)
2.  [Características Clave](#características-clave)
3.  [Alta Disponibilidad (High Availability)](#alta-disponibilidad-high-availability)
4.  [Réplicas de Lectura (Read Replicas)](#réplicas-de-lectura-read-replicas)
5.  [Seguridad y Conectividad](#seguridad-y-conectividad)
6.  [🧪 Laboratorio Práctico (CLI-TDD)](#laboratorio-práctico-cli-tdd)
7.  [💡 Tips de Examen](#tips-de-examen)
8.  [✍️ Resumen](#resumen)
9.  [🔖 Firma](#firma)

---

### Descripción

**Cloud SQL** es el servicio de bases de datos relacionales totalmente gestionado de Google Cloud. Automatiza todas las tareas tediosas de administración de bases de datos, como el aprovisionamiento, la aplicación de parches, las copias de seguridad y la configuración de la replicación, permitiéndote centrarte en tu aplicación.

Cloud SQL es compatible con los motores de bases de datos más populares: **MySQL, PostgreSQL y SQL Server**.

### Características Clave

*   **Totalmente Gestionado:** Google se encarga del sistema operativo, la instalación de la base de datos, los parches de seguridad y las actualizaciones.
*   **Copias de Seguridad Automatizadas:** Realiza copias de seguridad diarias automáticas y también permite copias de seguridad bajo demanda. Permite la **recuperación a un punto en el tiempo (Point-in-Time Recovery - PITR)** gracias a los registros binarios.
*   **Escalabilidad Sencilla:** Puedes escalar verticalmente tu instancia (aumentar vCPU y RAM) con un solo clic y un breve tiempo de inactividad.
*   **Seguridad Integrada:** Los datos se cifran en reposo y en tránsito. El acceso se controla a través de redes autorizadas y el Cloud SQL Auth Proxy.

### Alta Disponibilidad (High Availability)

La configuración de alta disponibilidad (HA) de Cloud SQL proporciona tolerancia a fallos a nivel de zona.

*   **¿Cómo funciona?**
    1.  Creas una instancia principal (master) en una zona.
    2.  Cloud SQL aprovisiona automáticamente una instancia **en espera (standby)** idéntica en una zona diferente dentro de la misma región.
    3.  Los datos se replican de forma **síncrona** en el disco persistente de ambas instancias.
    4.  Si la instancia principal deja de responder, Cloud SQL realiza una **conmutación por error (failover)** automática a la instancia en espera. La aplicación se redirige a la instancia en espera, que se convierte en la nueva principal.
*   **SLA:** Ofrece un SLA del 99.95%.
*   **Costo:** Pagas por los recursos de ambas instancias, la principal y la de espera.

### Réplicas de Lectura (Read Replicas)

Las réplicas de lectura se utilizan para escalar las cargas de trabajo de lectura, liberando a la instancia principal para que se encargue de las escrituras.

*   **¿Cómo funciona?**
    1.  Creas una o más réplicas de lectura a partir de una instancia principal.
    2.  Los datos se copian de forma **asíncrona** desde la principal a las réplicas.
    3.  Puedes dirigir todo tu tráfico de lectura (consultas `SELECT`) a las réplicas, distribuyendo la carga.
*   **Diferencia con HA:** Una réplica de lectura no proporciona conmutación por error automática. Es para escalar, no para alta disponibilidad. La replicación asíncrona significa que puede haber un pequeño retraso (lag) entre la principal y la réplica.
*   **Caso de Uso:** Aplicaciones con mucho tráfico de lectura, como paneles de business intelligence o sitios de contenido.

### Seguridad y Conectividad

*   **IP Privada:** La mejor práctica es configurar las instancias de Cloud SQL para que solo tengan una IP privada dentro de tu VPC. Esto evita cualquier exposición a la Internet pública.
*   **Redes Autorizadas:** Puedes configurar una lista de rangos de IP (CIDR) que tienen permiso para conectarse a tu instancia de Cloud SQL (si usas IP pública).
*   **Cloud SQL Auth Proxy:** Es la herramienta recomendada para conectarse a Cloud SQL, especialmente desde fuera de GCP. Es un pequeño cliente que se ejecuta en tu máquina local o en una VM. Crea un túnel seguro y cifrado hacia tu instancia de Cloud SQL, gestionando la autenticación y autorización a través de credenciales de IAM, sin necesidad de gestionar certificados SSL o IPs estáticas.

## 💻 Laboratorio Práctico (CLI-TDD)

### Cloud SQL con IP Privada

En este laboratorio, crearemos una **Red VPC totalmente nueva** para aislar nuestra base de datos, garantizando la máxima seguridad y resolviendo el problema del *peering* atascado que vimos en la red por defecto.

### 💡 Nomenclatura del Proyecto

| Elemento | Nuevo Nombre | Explicación para el Video |
| :--- | :--- | :--- |
| **Red VPC** | `red-maestra-sql` | Nuestra red privada y limpia. |
| **Instancia SQL** | `db-privada-pro` | El nombre de nuestra base de datos. |
| **Usuario** | `user_video` | El usuario de conexión. |
| **Base de Datos** | `datos_app_01` | El nombre de la base de datos interna. |
| **Rango de IP** | `rango-peering-pro` | El espacio que reservaremos para Google. |

-----

## 0\. 🛠️ Preparación: Crear una Red VPC Limpia

El problema que tuvimos con la red `default` es que el *peering* estaba atascado. Al crear una **Red VPC nueva**, nos aseguramos de que no haya ningún túnel de *peering* previo que nos bloquee.

```bash
# Crear la Red VPC (Red Privada)
# El flag --subnet-mode=custom nos permite crear subredes a medida.
gcloud compute networks create red-maestra-sql --subnet-mode=custom
```

> **Explicación:** Imagina la VPC como tu **casa privada en la nube**. Creamos una casa nueva para asegurarnos de que no tenga problemas de fontanería (peering atascado) de los inquilinos anteriores.

```bash
# Crear una subred dentro de nuestra Red VPC
# Elegimos una región y un rango de IPs para los recursos dentro de la red.
gcloud compute networks subnets create subred-maestra-sql \
    --network=red-maestra-sql \
    --region=us-central1 \
    --range=10.10.0.0/20
```

> **Explicación:** La subred es como el **piso dentro de tu casa**. Es donde pondremos nuestra base de datos. Usamos un rango IP interno (`10.10.0.0/20`) que solo es visible dentro de tu casa.

-----

## 1\. 🤝 Prerrequisitos: Private Service Access (Peering)

Ahora crearemos la conexión de *peering* en nuestra **nueva red limpia**.

```bash
# 1. Habilitar la API de Service Networking (siempre es el primer paso)
gcloud services enable servicenetworking.googleapis.com
```

> **Explicación:** Le decimos a Google que vamos a usar la función que permite conectar tu red privada con sus servicios gestionados (Cloud SQL).

```bash
# 2. Reservar un rango de IPs para Google en la NUEVA VPC
gcloud compute addresses create rango-peering-pro \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=red-maestra-sql
```

> **Explicación:** Esto es como **reservar un número de teléfono exclusivo** para que Google te llame. Este rango (`/16`) es solo para sus servicios y no puede ser usado por tus máquinas.

```bash
# 3. Crear la conexión de peering (¡El túnel!)
# Conectamos Google Services a nuestra nueva red VPC.
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=rango-peering-pro \
    --network=red-maestra-sql
```

> **Explicación:** Este comando construye el **túnel privado y seguro (el *peering*)** entre el centro de datos de Cloud SQL y tu nueva casa (`red-maestra-sql`).

-----

## 2\. ☁️ Creación de la Instancia de Cloud SQL

Ahora creamos la base de datos y la ponemos en nuestra nueva red privada.

```bash
# Crear la instancia de PostgreSQL, pequeña y sin IP pública
gcloud sql instances create db-privada-pro \
    --database-version=POSTGRES_13 \
    --region=us-central1 \
    --tier=db-f1-micro \
    --no-assign-ip \
    --network=red-maestra-sql
```

> **Explicación:** Creamos la base de datos y, usando **`--no-assign-ip`**, nos aseguramos de que solo tenga una **IP PRIVADA**. Esto es crucial para la seguridad, ya que **nadie desde internet puede acceder directamente**. El `network` apunta a nuestra casa nueva y limpia.

```bash
# Verificación de IP: Confirma que no hay IP pública (publicAddress debe ser vacío)
gcloud sql instances describe db-privada-pro --format='value(ipAddresses)'
```

> **Explicación:** Revisamos la matrícula de nuestra base de datos para confirmar que solo tiene una dirección interna y que la dirección pública (para internet) está vacía.

-----

## 3\. 🔑 Configuración de Base de Datos y Usuario

```bash
# Crear la base de datos de prueba
gcloud sql databases create datos_app_01 --instance=db-privada-pro

# Crear el usuario y contraseña simples (¡Solo para el lab!)
gcloud sql users create user_video --instance=db-privada-pro --password="my-password-1234"

echo "✅ Base de datos 'datos_app_01' y usuario 'user_video' creados."
```

> **Explicación:** Ponemos una puerta a nuestra base de datos (el usuario) y creamos el primer archivo (la base de datos) donde guardaremos información.

-----

## 4\. ✅ Prueba Final: Conexión Exitosa

Este es el momento de la verdad, donde probamos la conexión con IP privada.

```bash
# Conectarse a la instancia (pedirá la contraseña 'my-password-1234')
# El comando gcloud sql connect usa el Cloud SQL Auth Proxy automáticamente a través del túnel.
gcloud sql connect db-privada-pro --user=user_video --database=datos_app_01
```

> **Explicación:** Usamos **`gcloud sql connect`**. Este comando es mágico: automáticamente usa el **Cloud SQL Auth Proxy** (preinstalado en Cloud Shell) para viajar por el túnel privado que creamos y conectarse a la base de datos. **¡Si el *peering* funciona, esto se conecta\!**

-----

### 5\. 🗑️ Limpieza (Para Evitar Cargos)

```bash
# 1. Eliminar la instancia de Cloud SQL (¡el paso que siempre debe ir primero!)
gcloud sql instances delete db-privada-pro --quiet

# 2. Eliminar el Peering VPC (Debería funcionar ahora porque la instancia ya no lo usa)
gcloud services vpc-peerings delete servicenetworking-googleapis-com \
    --network=red-maestra-sql \
    --service=servicenetworking.googleapis.com \
    --quiet

# 3. Eliminar el Rango de IPs reservado
gcloud compute addresses delete rango-peering-pro --global --quiet

# 4. Eliminar la Red VPC (la casa) y la subred
gcloud compute networks subnets delete subred-maestra-sql --region=us-central1 --quiet
gcloud compute networks delete red-maestra-sql --quiet
```

> **Explicación:** El **orden** es crucial: **Instancia \> Peering \> IP Reservada \> Red**. Si sigues este orden, evitas los errores de "recurso en uso" que tuvimos antes.

-----

## 💥 Resumen de Errores Vistos (Para el Video)

| Error Visto | Comando Fallido | Causa Real | Solución |
| :--- | :--- | :--- | :--- |
| `ERROR: ... database instance does not have an ipv4 address.` | `gcloud sql connect` | **El peering VPC no estaba activo.** No encontró la ruta. | Crear una **nueva VPC** (Paso 0) y el *peering*. |
| `ERROR: Cannot modify allocated ranges in CreateConnection...` | `gcloud services vpc-peerings connect` | **El peering ya existía** en la red `default`. | Usar el comando **`delete`** primero o **crear una VPC nueva** (como hicimos aquí). |
| `ERROR: Failed to delete connection; Producer services... are still using this connection.` | `gcloud services vpc-peerings delete` | **Una instancia de Cloud SQL seguía activa** (o el sistema de Google tenía un retraso). | **Eliminar TODAS las instancias de SQL** y esperar 5-10 minutos antes de borrar el *peering*. |

### 💡 Tips de Examen

*   **Cloud SQL vs. Spanner:** Si la pregunta habla de una base de datos relacional para una aplicación **regional** (como un CRM o un blog de WordPress), la respuesta es **Cloud SQL**. Si menciona **escalabilidad horizontal global** y **consistencia estricta**, es **Spanner**.
*   **HA vs. Réplicas de Lectura:** Si el objetivo es la **tolerancia a fallos** y la **recuperación automática**, la solución es la **Alta Disponibilidad (HA)**. Si el objetivo es **escalar el rendimiento de las lecturas**, la solución son las **Réplicas de Lectura**.
*   **Conectividad Segura:** La forma recomendada y más segura de conectarse a Cloud SQL es usando **IP Privada** y el **Cloud SQL Auth Proxy**.

### ✍️ Resumen

Cloud SQL es la solución ideal para cargas de trabajo relacionales en GCP que no requieren la escala masiva de Spanner. Al ser un servicio totalmente gestionado, elimina la carga operativa de la administración de bases de datos. Sus funcionalidades integradas de alta disponibilidad, réplicas de lectura, copias de seguridad automáticas y conectividad segura a través del Auth Proxy lo convierten en una opción robusta y fácil de usar para la mayoría de las aplicaciones tradicionales.

---

## ✍️ Firma

**Marco - DevSecOps Kulture**  
*The Artisan Path*  
📧 Contacto: [markitos.es.info@gmail.com](mailto:markitos.es.info@gmail.com)  
🐙 GitHub: [https://github.com/markitos-public](https://github.com/markitos-public)

---

[⬆️ **Volver arriba**](#-004-cloud-sql)
