
# 📜 001: Compatibilidades Importantes de VPC

## 📝 Índice

1.  [Descripción](#descripción)
2.  [VPC Peering (Interconexión de VPC)](#vpc-peering-interconexión-de-vpc)
3.  [Shared VPC (VPC Compartida)](#shared-vpc-vpc-compartida)
4.  [Private Google Access (Acceso Privado a Google)](#private-google-access-acceso-privado-a-google)
5.  [Alias IP Ranges (Rangos de IP de Alias)](#alias-ip-ranges-rangos-de-ip-de-alias)
6.  [VPC Flow Logs (Registros de Flujo de VPC)](#vpc-flow-logs-registros-de-flujo-de-vpc)
7.  [🧪 Laboratorio Práctico (CLI-TDD)](#laboratorio-práctico-cli-tdd)
8.  [💡 Tips de Examen](#tips-de-examen)
9.  [✍️ Resumen](#resumen)
10. [🔖 Firma](#firma)

---

### Descripción

Una VPC por sí sola es potente, pero su verdadero poder se desbloquea al conectarla y extenderla con otros servicios de red de GCP. Estas "compatibilidades" o funcionalidades adicionales permiten crear arquitecturas de red complejas, seguras y administrables que abarcan múltiples proyectos y se conectan de forma segura a los servicios de Google o a entornos on-premise.

Este capítulo cubre las formas más importantes de extender y administrar tu red de VPC.

### VPC Peering (Interconexión de VPC)

*   **¿Qué es?** Permite conectar dos redes VPC de forma privada y directa, sin usar la Internet pública. Las VMs en las VPCs interconectadas pueden comunicarse usando sus IPs internas.
*   **Características Clave:**
    *   **No Transitiva:** Es la limitación más importante. Si la VPC-A está conectada con la VPC-B, y la VPC-B con la VPC-C, la VPC-A **no puede** comunicarse con la VPC-C a través de la B. Se necesitaría una conexión directa entre A y C.
    *   **Descentralizada:** Cada proyecto administra su propio lado de la conexión de peering.
    *   **Rangos de IP:** Los rangos de subred de las VPCs interconectadas no deben solaparse.
*   **Caso de Uso:** Conectar VPCs de diferentes proyectos o equipos que necesitan comunicarse de forma privada. Por ejemplo, un proyecto de frontend que necesita acceder a un backend en otro proyecto.

### Shared VPC (VPC Compartida)

*   **¿Qué es?** Permite a una organización designar un proyecto como **Proyecto Host**, que posee y administra una o más redes VPC. Otros proyectos, llamados **Proyectos de Servicio**, pueden usar las subredes de esa VPC compartida para desplegar sus recursos (VMs, GKE, etc.).
*   **Características Clave:**
    *   **Centralizada:** La administración de la red (firewalls, subredes, rutas) se centraliza en el Proyecto Host, promoviendo la consistencia y la seguridad.
    *   **Roles de IAM:** Se basa en roles específicos:
        *   `roles/compute.xpnAdmin` (Administrador de VPC Compartida): Se otorga a nivel de Organización para designar Proyectos Host.
        *   `roles/compute.networkUser` (Usuario de Red): Se otorga a los miembros de los Proyectos de Servicio para que puedan usar las subredes compartidas.
*   **Caso de Uso:** Gobernanza de red en organizaciones grandes. Un equipo de red centralizado gestiona la infraestructura de red, y los equipos de desarrollo consumen esa red en sus propios proyectos.

### Private Google Access (Acceso Privado a Google)

*   **¿Qué es?** Permite que las VMs de tu VPC que **solo tienen IPs internas** accedan a las APIs y servicios de Google (como Cloud Storage, BigQuery, etc.) sin necesidad de una IP externa.
*   **Características Clave:**
    *   Se habilita por subred.
    *   Utiliza un conjunto especial de dominios y VIPs (Virtual IP) que Google proporciona para este fin (ej. `private.googleapis.com`).
*   **Caso de Uso:** Mejorar la seguridad al permitir que las VMs de backend operen sin una IP pública, pero aún puedan descargar dependencias, acceder a APIs de Google o guardar datos en Cloud Storage.

### Alias IP Ranges (Rangos de IP de Alias)

*   **¿Qué es?** Permite asignar rangos de direcciones IP internas de una subred como IPs secundarias a la interfaz de red de una VM. Es decir, una VM puede tener múltiples IPs del mismo rango de subred.
*   **Características Clave:**
    *   Es la base del networking de Google Kubernetes Engine (GKE). Cada Pod en GKE obtiene su propia IP de un rango de alias, permitiendo la comunicación directa sin NAT.
*   **Caso de Uso:** Principalmente para GKE (VPC-native clusters). También es útil para aplicaciones que necesitan alojar múltiples servicios en una sola VM, cada uno vinculado a una IP diferente.

### VPC Flow Logs (Registros de Flujo de VPC)

*   **¿Qué es?** Es una herramienta de telemetría que captura información sobre el tráfico IP que entra y sale de las interfaces de red de las VMs.
*   **Características Clave:**
    *   Se habilita por subred.
    *   No captura el contenido del tráfico, solo metadatos (IPs de origen/destino, puertos, protocolo, bytes transferidos, etc.).
    *   Los registros se envían a Cloud Logging, donde se pueden analizar.
*   **Caso de Uso:** Monitorización de red, análisis de seguridad (forense), optimización de rendimiento y costos.

### 🧪 Laboratorio Práctico (CLI-TDD)

**Objetivo:** Crear una interconexión (peering) entre dos VPCs. La conexión debe establecerse en ambas direcciones (de A a B y de B a A) porque el VPC Peering es una relación de consentimiento mutuo; ambas redes deben aceptar explícitamente la conexión para que el tráfico pueda fluir. Si solo se configura en una dirección (por ejemplo, de A a B), la conexión permanecerá en estado `INACTIVE` y no se intercambiarán rutas, por lo que la comunicación entre las VPCs no será posible.

```bash
# 1. Crear dos VPCs
gcloud compute networks create vpc-a --subnet-mode=custom
gcloud compute networks create vpc-b --subnet-mode=custom
gcloud compute networks subnets create subnet-a --network=vpc-a --range=10.0.1.0/24 --region=us-central1
gcloud compute networks subnets create subnet-b --network=vpc-b --range=10.0.2.0/24 --region=us-central1

# 2. Crear la conexión de peering de A hacia B
gcloud compute networks peerings create peering-a-to-b \
    --network=vpc-a \
    --peer-network=vpc-b

# 3. Crear la conexión de peering de B hacia A
gcloud compute networks peerings create peering-b-to-a \
    --network=vpc-b \
    --peer-network=vpc-a

# 4. Test (Verificación): Listar los peerings de una de las VPCs
gcloud compute networks peerings list --network=vpc-a
# Esperado: Debería mostrar una entrada para 'peering-a-to-b' con estado 'ACTIVE'.
# Esto confirma que la interconexión está activa y las rutas se han intercambiado.
```

### 💡 Tips de Examen

*   **Shared VPC vs. VPC Peering:** Si la pregunta habla de **administración centralizada** y **gobernanza** en una organización, la respuesta es **Shared VPC**. Si habla de conectar redes de **proyectos separados** que necesitan comunicarse, es **VPC Peering**.
*   **Transitive Peering:** Recuerda siempre la limitación de que el peering **no es transitivo**. Es un escenario de fallo muy común en las preguntas.
*   **Private Google Access:** Si una VM sin IP externa necesita acceder a APIs de Google, esta es la solución.

### ✍️ Resumen

Las funcionalidades de compatibilidad de VPC transforman una simple red virtual en una plataforma de networking robusta y gestionable. **VPC Peering** y **Shared VPC** resuelven la comunicación y gobernanza entre proyectos. **Private Google Access** mejora la seguridad al eliminar la necesidad de IPs externas para acceder a servicios de Google. **Alias IP** es fundamental para la contenerización moderna con GKE, y **VPC Flow Logs** proporciona la visibilidad necesaria para operar y asegurar la red.

---

## ✍️ Firma

**Marco - DevSecOps Kulture**  
*The Artisan Path*  
📧 Contacto: [markitos.es.info@gmail.com](mailto:markitos.es.info@gmail.com)  
🐙 GitHub: [https://github.com/markitos-public](https://github.com/markitos-public)

---

[⬆️ **Volver arriba**](#-001-compatibilidades-importantes-de-vpc)
