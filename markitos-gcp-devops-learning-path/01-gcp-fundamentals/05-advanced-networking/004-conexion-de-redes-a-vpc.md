
# 📜 004: Conexión de Redes a VPC (Conectividad Híbrida)

## 📝 Índice

1.  [Descripción](#descripción)
2.  [El Desafío de la Conectividad Híbrida](#el-desafío-de-la-conectividad-híbrida)
3.  [Cloud VPN](#cloud-vpn)
4.  [Cloud Interconnect](#cloud-interconnect)
5.  [Network Connectivity Center](#network-connectivity-center)
6.  [Tabla Comparativa](#tabla-comparativa)
7.  [🧪 Laboratorio Práctico (CLI-TDD)](#laboratorio-práctico-cli-tdd)
8.  [💡 Tips de Examen](#tips-de-examen)
9.  [✍️ Resumen](#resumen)
10. [🔖 Firma](#firma)

---

### Descripción

La **conectividad híbrida** se refiere a la conexión de tu red on-premise (tu centro de datos corporativo) con tu red de nube privada virtual (VPC) en Google Cloud. Esta conexión es fundamental para la mayoría de las empresas, ya que les permite migrar cargas de trabajo gradualmente, crear arquitecturas que abarcan ambos entornos y permitir que los recursos en la nube accedan a datos o servicios on-premise de forma segura.

GCP ofrece un conjunto de servicios para establecer esta conectividad, principalmente **Cloud VPN** y **Cloud Interconnect**.

### El Desafío de la Conectividad Híbrida

El objetivo es extender tu red corporativa a GCP de forma segura y fiable. Las principales consideraciones al elegir una solución son:

*   **Ancho de banda:** ¿Cuántos datos necesitas transferir?
*   **Latencia:** ¿Qué tan sensible es tu aplicación a los retrasos de la red?
*   **Fiabilidad y SLA:** ¿Qué nivel de disponibilidad necesitas?
*   **Costo:** ¿Cuál es tu presupuesto?

### Cloud VPN

*   **¿Qué es?** Establece una conexión segura entre tu red on-premise y tu VPC a través de un **túnel IPsec** que viaja por la Internet pública. Es la forma más rápida y sencilla de empezar con la conectividad híbrida.
*   **Tipos:**
    1.  **HA VPN (High Availability VPN):** La solución recomendada para producción. Crea un par de túneles redundantes con un SLA del 99.99%. Requiere configurar dos túneles en tu dispositivo VPN on-premise. Utiliza enrutamiento dinámico con BGP (Border Gateway Protocol).
    2.  **Classic VPN:** La versión anterior. Generalmente crea un solo túnel con un SLA del 99.9%. Admite enrutamiento estático y dinámico. Se considera heredada y se debe preferir HA VPN para nuevas implementaciones.
*   **Caso de Uso:** Cargas de trabajo con requisitos de ancho de banda de bajos a moderados (hasta varios Gbps). Ideal para empezar, para entornos de desarrollo/pruebas, o como respaldo de una conexión de Interconnect.

#### Arquitectura de HA VPN para Alta Disponibilidad

La garantía del SLA del 99.99% de HA VPN no es magia, sino el resultado de una arquitectura redundante y dinámica.

*   **Componentes:** Una puerta de enlace de HA VPN en GCP tiene **dos interfaces**, cada una con su propia dirección IP externa. Debes configurar **dos túneles VPN** desde tu puerta de enlace on-premise, uno hacia cada una de las interfaces de la puerta de enlace de GCP.

*   **Topología Activo-Activo:** Ambos túneles están siempre activos. No es una configuración activo-pasivo. El tráfico puede fluir por ambos túneles simultáneamente.

*   **Enrutamiento Dinámico con BGP:** Aquí está la clave del failover automático. Se establece una sesión BGP sobre cada túnel. A través de BGP, los routers de ambos lados (GCP y on-premise) intercambian y aprenden las rutas de red. Si un túnel falla (por un problema de red o en el hardware), la sesión BGP de ese túnel se cae. El router BGP se da cuenta inmediatamente de que las rutas a través de ese túnel ya no son válidas y **automáticamente desvía todo el tráfico al segundo túnel**, que permanece activo. Este proceso es automático y tarda solo unos segundos, garantizando una interrupción mínima o nula.

*   **Redundancia Completa:** Para una verdadera resiliencia de extremo a extremo, la topología recomendada implica tener dos puertas de enlace VPN también en el lado on-premise, con cada una conectándose a la puerta de enlace de HA VPN de GCP. Esto protege contra fallos tanto en la red de GCP como en tu propio hardware.

### Cloud Interconnect

*   **¿Qué es?** Proporciona una conexión física y privada de baja latencia y alta disponibilidad entre tu red on-premise y la red de Google. El tráfico **no viaja por la Internet pública**, lo que ofrece mayor rendimiento y seguridad.
*   **Tipos:**
    1.  **Dedicated Interconnect:**
        *   **Concepto:** Obtienes una conexión física directa (un puerto) a la red de Google en una ubicación de coubicación (colocation facility).
        *   **Ancho de banda:** Circuitos de 10 Gbps o 100 Gbps.
        *   **SLA:** Hasta 99.99% con configuración redundante.
        *   **Caso de Uso:** Cargas de trabajo a gran escala que necesitan transferir terabytes de datos con un rendimiento constante y baja latencia.

    2.  **Partner Interconnect:**
        *   **Concepto:** Te conectas a la red de Google a través de un proveedor de servicios asociado. Es más flexible si no te encuentras en una de las ubicaciones de Dedicated Interconnect.
        *   **Ancho de banda:** Conexiones desde 50 Mbps hasta 50 Gbps.
        *   **SLA:** Depende del proveedor, pero puede llegar al 99.99%.
        *   **Caso de Uso:** Empresas que necesitan una conexión privada pero no requieren un circuito dedicado de 10 Gbps, o que prefieren la flexibilidad de un proveedor.

### Network Connectivity Center

*   **¿Qué es?** Es un servicio de gestión centralizada que utiliza la red troncal de Google para conectar tus diferentes redes empresariales (VPCs, VPNs, Interconnects, redes de terceros) en un modelo de **hub-and-spoke**.
*   **Caso de Uso:** Simplificar la gestión de redes complejas a gran escala, permitiendo que diferentes sitios on-premise se comuniquen entre sí a través de la red de Google, en lugar de tener que enrutar el tráfico a través de tu centro de datos principal.

### Tabla Comparativa

| Característica      | HA VPN                                  | Dedicated Interconnect                  | Partner Interconnect                    |
| ------------------- | --------------------------------------- | --------------------------------------- | --------------------------------------- |
| **Medio**           | Internet Pública (cifrado)              | Fibra privada directa a Google          | Fibra privada a través de un partner    |
| **Ancho de Banda**  | Moderado (Gbps por túnel)               | Muy Alto (10/100 Gbps)                  | Flexible (50 Mbps - 50 Gbps)            |
| **Latencia**        | Variable                                | Muy baja y predecible                   | Baja y predecible                       |
| **SLA**             | 99.99%                                  | 99.99%                                  | Hasta 99.99% (depende del partner)      |
| **Caso de Uso**     | Entornos de dev/test, backup, bajo tráfico | Cargas masivas de datos, alta demanda   | Conectividad privada flexible           |

### 🧪 Laboratorio 1: Crear un Túnel de Classic VPN (Ejemplo Didáctico)

**Objetivo:** Crear una conexión VPN básica entre una VPC de GCP y una red "on-premise" simulada.
**Nota:** Este laboratorio usa Classic VPN con una ruta estática por simplicidad. Para entornos de producción, Google recomienda encarecidamente usar **HA VPN**.

```bash
# --- Explicación de los Conceptos Clave ---
# [GCP_VPN_GATEWAY_IP]: Es la IP pública para la puerta de enlace de VPN en GCP. No necesitas tener este valor de antemano. El **Paso 1** del script se encarga de reservar una nueva dirección IP estática y la asigna a una variable para que los siguientes pasos la usen.
# [PEER_IP]: Es la IP pública de nuestro router VPN "on-premise". GCP la necesita para saber a dónde conectarse.
# [ON_PREM_RANGE]: Es el bloque de direcciones IP de la red "on-premise" (ej: 192.168.1.0/24). GCP necesita saber esto para enrutar el tráfico correctamente a través del túnel.
# [SHARED_SECRET]: Es una "contraseña" secreta que ambos lados del túnel deben conocer para establecer una conexión segura.

# --- Variables de Configuración ---
# ¡Reemplaza estos valores!
export GCP_REGION="us-central1"
export GCP_VPC_NETWORK="default"

# Para [ON_PREM_PEER_IP], en un escenario real, usarías la IP pública de tu router VPN.
# Para este lab, puedes usar la IP pública de tu propia máquina para simularlo.
# Búscala, por ejemplo, en https://ifconfig.me
export ON_PREM_PEER_IP="[TU_IP_PÚBLICA_AQUÍ]"

# Este es el rango de IPs de tu red "on-premise" simulada.
export ON_PREM_IP_RANGE="192.168.1.0/24"

# Este es un secreto compartido que debes generar. Debe ser una cadena segura.
export VPN_SHARED_SECRET="tu-secreto-super-seguro-aqui-12345"

# --- Pasos ---

# 1. Reservar una dirección IP estática para la puerta de enlace de Cloud VPN
echo "Paso 1: Creando IP estática para la puerta de enlace VPN..."
gcloud compute addresses create gcp-vpn-gateway-ip --region=${GCP_REGION}
export GCP_VPN_GATEWAY_IP=$(gcloud compute addresses describe gcp-vpn-gateway-ip --region=${GCP_REGION} --format='value(address)')
echo "--> Puerta de enlace VPN usará la IP: ${GCP_VPN_GATEWAY_IP}"

# 2. Crear la puerta de enlace de Cloud VPN (target-vpn-gateway) en GCP
echo "Paso 2: Creando la puerta de enlace de Classic VPN en GCP..."
gcloud compute target-vpn-gateways create gcp-vpn-gateway --network=${GCP_VPC_NETWORK} --region=${GCP_REGION}

# 3. Crear las reglas de reenvío (forwarding rules)
# Se necesitan tres reglas de reenvío para dirigir el tráfico IPsec a la puerta de enlace VPN:
# - IPsec ESP (Protocolo 50): Cifra y autentica los datos del paquete (el payload).
# - UDP 500: Para el intercambio inicial de claves (IKE) que establece el túnel seguro.
# - UDP 4500: Para el NAT Traversal (NAT-T), que encapsula el tráfico IPsec en UDP para que pueda atravesar routers que usan NAT.
echo "Paso 3: Creando reglas de reenvío..."
gcloud compute forwarding-rules create gcp-vpn-rule-esp --region=${GCP_REGION} \
    --ip-protocol=ESP --address=${GCP_VPN_GATEWAY_IP} --target-vpn-gateway=gcp-vpn-gateway

gcloud compute forwarding-rules create gcp-vpn-rule-udp500 --region=${GCP_REGION} \
    --ip-protocol=UDP --ports=500 --address=${GCP_VPN_GATEWAY_IP} --target-vpn-gateway=gcp-vpn-gateway

gcloud compute forwarding-rules create gcp-vpn-rule-udp4500 --region=${GCP_REGION} \
    --ip-protocol=UDP --ports=4500 --address=${GCP_VPN_GATEWAY_IP} --target-vpn-gateway=gcp-vpn-gateway

# 4. Crear el túnel VPN
# Aquí es donde conectamos nuestra puerta de enlace con la IP del par "on-premise".
echo "Paso 4: Creando el túnel VPN..."
gcloud compute vpn-tunnels create gcp-to-onprem-tunnel --region=${GCP_REGION} \
    --peer-ip-address=${ON_PREM_PEER_IP} \
    --target-vpn-gateway=gcp-vpn-gateway \
    --ike-version=2 \
    --shared-secret="${VPN_SHARED_SECRET}"

# 5. Crear una ruta estática
# Esto le dice a la VPC cómo llegar a la red on-premise a través del túnel que creamos.
echo "Paso 5: Creando ruta estática hacia la red on-premise..."
gcloud compute routes create route-to-on-prem --network=${GCP_VPC_NETWORK} \
    --destination-range=${ON_PREM_IP_RANGE} \
    --next-hop-vpn-tunnel=gcp-to-onprem-tunnel \
    --next-hop-vpn-tunnel-region=${GCP_REGION}

# 6. Verificar el Estado del Túnel
# Este comando te mostrará el estado del túnel en el lado de GCP.
# Al principio, estará en estados como "WAITING_FOR_TRAFFIC" o "FIRST_HANDSHAKE"
# porque todavía falta configurar el otro extremo (on-premise).
# Una vez configurado el par, el estado debería cambiar a "ESTABLISHED".
echo "Paso 6: Verificando el estado del túnel VPN (lado de GCP)..."
gcloud compute vpn-tunnels describe gcp-to-onprem-tunnel --region=${GCP_REGION} --format='table(name,region,status,detailedStatus)'

echo "¡Configuración de Classic VPN en GCP completada!"
echo "El siguiente paso crítico es configurar tu router 'on-premise' con la IP ${GCP_VPN_GATEWAY_IP} y el secreto compartido."
echo "Usa el comando del Paso 6 para monitorear el estado hasta que veas 'ESTABLISHED'."
```

### 🧪 Laboratorio 2: Conexión Segura a una VM sin IP Pública con IAP

**Objetivo:** Conectarse de forma segura (SSH) a una VM que no tiene IP pública, utilizando el túnel TCP de IAP (Identity-Aware Proxy).
**Este es el método recomendado por Google para el acceso de administradores**, ya que no expone las VMs a Internet.

```bash
# --- Requisitos Previos ---
# 1. La API de IAP debe estar habilitada en tu proyecto.
#    Puedes habilitarla con: gcloud services enable iap.googleapis.com

# --- Variables de Configuración ---
# El script obtiene automáticamente los valores necesarios de tu configuración de gcloud.
export GCP_PROJECT_ID=$(gcloud config get-value project)
export GCP_ACCOUNT=$(gcloud config get-value account)
export GCP_ZONE="us-central1-a" # Puedes cambiar esto a tu zona preferida

echo "-> Usando el proyecto: ${GCP_PROJECT_ID}"
echo "-> Usando la cuenta: ${GCP_ACCOUNT}"
echo "-> Usando la zona: ${GCP_ZONE}"

# --- Pasos ---

# Paso 1: Crear una VM de prueba SIN dirección IP pública
# El flag --no-address es la clave aquí. Esto asegura que la VM no sea accesible desde Internet.
echo "Paso 1: Creando VM 'vm-privada-iap' sin IP pública..."
gcloud compute instances create vm-privada-iap --zone=${GCP_ZONE} \
    --machine-type=e2-micro \
    --image-family=debian-11 --image-project=debian-cloud \
    --no-address

# Paso 2: Crear una regla de firewall para permitir el acceso DESDE IAP
# Esta regla permite que el servicio de IAP (cuyas IPs están en el rango 35.235.240.0/20)
# se conecte al puerto SSH (22) de las VMs en tu red. No abre el puerto a todo Internet.
echo "Paso 2: Creando regla de firewall para permitir conexiones SSH desde IAP..."
gcloud compute firewall-rules create allow-ssh-via-iap --network=default \
    --allow=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --description="Permitir conexiones SSH entrantes solo desde el servicio IAP de Google"

# Paso 3: Otorgar a tu usuario el permiso para USAR el túnel de IAP
# Este rol de IAM ('IAP-secured Tunnel User') permite a tu cuenta crear un túnel
# seguro hacia la VM. No da permisos de administrador en la VM, solo el permiso
# para conectarse a través de IAP.
echo "Paso 3: Otorgando a ${GCP_ACCOUNT} el rol para usar túneles IAP..."
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
    --member="user:${GCP_ACCOUNT}" \
    --role="roles/iap.tunnelResourceAccessor"

echo "Esperando 10 segundos para que los permisos de IAM se propaguen..."
sleep 10

# Paso 4: Conectarse a la VM privada a través de IAP
# gcloud se encarga de crear el túnel de IAP de forma transparente.
# Simplemente usa el comando SSH normal. gcloud detecta que no puede conectar
# directamente y usará el túnel IAP automáticamente.
echo "Paso 4: Intentando conectar a la VM vía SSH a través de IAP..."
gcloud compute ssh vm-privada-iap --zone=${GCP_ZONE}

# ¡Si todo ha ido bien, estarás dentro de la VM sin que esta tenga IP pública!

# --- Verificación (Dentro de la VM) ---
# Una vez dentro, puedes verificar que no tienes acceso a internet (a menos que tengas Cloud NAT).
# ping google.com
# El comando debería fallar, demostrando que la VM está aislada.

# --- Limpieza (Opcional pero Recomendado) ---
# echo "Limpiando los recursos creados..."
# gcloud compute instances delete vm-privada-iap --zone=${GCP_ZONE} --quiet
# gcloud compute firewall-rules delete allow-ssh-via-iap --quiet
# gcloud projects remove-iam-policy-binding ${GCP_PROJECT_ID} --member="user:${GCP_ACCOUNT}" --role="roles/iap.tunnelResourceAccessor" --quiet
# gcloud compute addresses delete gcp-vpn-gateway-ip --region=${GCP_REGION} --quiet
```

### 🧪 Laboratorio 3: Cloud Interconnect (Conceptual)

**Objetivo:** Entender por qué no podemos replicar una configuración de Cloud Interconnect en un laboratorio simple y cuáles serían los pasos en un escenario real.

A diferencia de Cloud VPN, que se configura lógicamente sobre la Internet pública, **Cloud Interconnect es un servicio físico**. Requiere establecer una conexión de fibra óptica privada entre tu red on-premise y la red de Google, lo cual implica procesos que no se pueden simular con comandos `gcloud`:

1.  **Contacto y Contratación:**
    *   **Dedicated Interconnect:** Implica solicitar una "cross-connect" (interconexión física) en un centro de datos (colocation facility) donde tanto tú como Google tengáis presencia.
    *   **Partner Interconnect:** Implica contratar el servicio a través de un proveedor de telecomunicaciones asociado.

2.  **Instalación Física:** Un técnico debe instalar físicamente el cableado de fibra para conectar tu equipamiento al del proveedor o al de Google. Este proceso puede tardar semanas o meses.

3.  **Configuración de Red Avanzada:** Una vez establecida la conexión física, la configuración se realiza en routers de borde de nivel empresarial. Implica configurar el protocolo de enrutamiento **BGP** para intercambiar rutas entre tu red y la red de Google.

**Conclusión del Laboratorio Conceptual:**
No es posible crear un script `gcloud` para "crear" una interconexión como lo hacemos con una VPN. La configuración en la consola de GCP (crear el "VLAN attachment") es solo el último paso de un proceso logístico y físico complejo. Entender esta diferencia es clave para el examen y la práctica profesional.

### 💡 Tips de Examen

*   **VPN vs. Interconnect:** Es la decisión más común en las preguntas. Si se menciona **Internet pública**, la respuesta es **VPN**. Si se habla de **conexión dedicada, privada o de baja latencia/alto ancho de banda**, la respuesta es **Interconnect**.
*   **Dedicated vs. Partner Interconnect:** Si la pregunta implica la necesidad de un circuito masivo de **10 Gbps o 100 Gbps** y estar en una ubicación de coubicación, es **Dedicated**. Si se necesita más **flexibilidad** en el ancho de banda o la ubicación, es **Partner**.
*   **HA VPN:** Si se requiere un SLA del **99.99%** sobre VPN, la respuesta es **HA VPN**.

### ✍️ Resumen

La conectividad híbrida es un pilar de la adopción de la nube en la empresa. Cloud VPN ofrece una solución rápida y segura sobre la Internet pública, ideal para empezar o para cargas de trabajo no intensivas. Cloud Interconnect (Dedicated y Partner) proporciona una conexión privada de nivel empresarial para las cargas de trabajo más exigentes en términos de rendimiento y fiabilidad. La elección entre estas opciones depende de un análisis cuidadoso de los requisitos de ancho de banda, latencia, seguridad y costo.

---

## ✍️ Firma

**Marco - DevSecOps Kulture**  
*The Artisan Path*  
📧 Contacto: [markitos.es.info@gmail.com](mailto:markitos.es.info@gmail.com)  
🐙 GitHub: [https://github.com/markitos-public](https://github.com/markitos-public)

---

[⬆️ **Volver arriba**](#-004-conexión-de-redes-a-vpc-conectividad-híbrida)

---

## 🎙️ Guion para Vídeo (Modo Podcast)

**(Inicio con música de fondo tecnológica y un tono de arquitecto de soluciones)**

¡Hola y bienvenidos a un nuevo capítulo práctico de DevSecOps Kulture! Hoy vamos a construir puentes. Puentes digitales, claro. Hablaremos de **conectividad híbrida**: cómo conectar de forma segura y fiable tu oficina o tu centro de datos con tu red privada en Google Cloud.

Imagina que tienes una base de datos en tu empresa que tu nueva aplicación en la nube necesita consultar. ¿Cómo lo haces? ¿Abres un puerto a Internet? ¡Por supuesto que no! Hoy te voy a enseñar los dos caminos principales para construir esa conexión como un profesional: Cloud VPN y Cloud Interconnect.

---

### Camino 1: Cloud VPN - El Túnel Blindado por Internet

**Tú:** "Necesito una conexión segura, pero no quiero gastar una fortuna. ¿Qué hago?"

**Yo:** Tu respuesta es **Cloud VPN**. Piensa en ello como construir un túnel blindado y secreto que viaja por la autopista pública de Internet. Es rápido de configurar, seguro y relativamente económico.

La opción que Google recomienda para cualquier entorno serio es la **HA VPN**, o VPN de Alta Disponibilidad. Te da un impresionante SLA del 99.99%.

**Tú:** "¿Y cómo logra esa disponibilidad?"

**Yo:** ¡No es magia, es redundancia! HA VPN te da **dos túneles** en lugar de uno. Ambos están activos todo el tiempo. Si uno de los túneles falla por cualquier motivo, el enrutamiento dinámico (gracias a un protocolo llamado BGP) se da cuenta al instante y desvía todo el tráfico por el segundo túnel. ¡El failover es automático y casi instantáneo!

En resumen: Cloud VPN es perfecto para empezar, para entornos de desarrollo, o para cargas de trabajo con un tráfico moderado.

---

### Camino 2: Cloud Interconnect - Tu Autopista Privada a la Nube

**Tú:** "Vale, pero yo muevo terabytes de datos. Necesito máximo rendimiento y una latencia súper baja y predecible."

**Yo:** Entonces, necesitas tu propia autopista privada. Eso es **Cloud Interconnect**. Con Interconnect, tu tráfico **nunca toca la Internet pública**. Es una conexión física y directa a la red de Google.

Aquí tienes dos sabores:

1.  **Dedicated Interconnect:** Esto es para los pesos pesados. Alquilas un puerto de fibra óptica de 10 o 100 Gigabits por segundo directamente en un centro de datos donde Google tiene presencia. Es el máximo rendimiento posible.

2.  **Partner Interconnect:** Esta es la opción más flexible y común. Te conectas a través de uno de los muchos socios de Google. Puedes contratar el ancho de banda que necesites, desde 50 Megabits hasta 50 Gigabits.

En resumen: si la latencia, el ancho de banda masivo y la seguridad de una red privada son tus prioridades, Cloud Interconnect es el camino.

---

### La Gran Decisión: ¿VPN o Interconnect?

La elección es simple si te haces las preguntas correctas:

*   ¿Tu conexión es a través de **Internet**? Es **VPN**.
*   ¿Es una **conexión física y privada**? Es **Interconnect**.
*   ¿Necesitas un SLA del **99.99%**? Puedes lograrlo con **HA VPN** o con una **Interconexión redundante**.
*   ¿Necesitas transferir **terabytes de datos** con rendimiento constante? La respuesta es **Interconnect**.

---

### Conclusión

Y ahí lo tienes. Conectar tu mundo on-premise con Google Cloud es una decisión estratégica. **Cloud VPN** te da una solución rápida y segura sobre Internet, mientras que **Cloud Interconnect** te ofrece una autopista privada de alto rendimiento. Elige la herramienta adecuada para el trabajo y construirás una arquitectura híbrida robusta y fiable.

**(Música de cierre)**

¡Gracias por acompañarnos! Si te ha gustado, no olvides suscribirte y dejar un comentario sobre los desafíos de red más complejos que has enfrentado. ¡Nos vemos en el próximo capítulo!
