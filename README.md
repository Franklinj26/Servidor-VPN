# Servidor-VPN
Vamos a crear un servidor VPN con el fin de emular como nos conectaríamos a la red de nuestra oficina de forma remota desde cualquier punto. 


# Configuración de VPN Site-to-Site y acceso remoto con pfSense y OpenVPN

## 📌 Descripción del proyecto

Este proyecto consiste en la configuración de dos conexiones VPN utilizando **pfSense Community Edition y OpenVPN**:

- **VPN Site-to-Site:** permite conectar dos redes locales diferentes para que los equipos de ambas redes puedan comunicarse entre sí.
- **VPN de acceso remoto:** permite que un usuario externo se conecte de forma segura a la red local del pfSense11 mediante un cliente OpenVPN.

La configuración se ha realizado utilizando máquinas virtuales y equipos Windows, creando certificados digitales, configurando los servidores y clientes OpenVPN y estableciendo las correspondientes reglas de firewall.

## 🛠️ Tecnologías utilizadas

- pfSense Community Edition.
- OpenVPN.
- OpenVPN Client Export.
- Windows 11.
- Certificados digitales X.509.
- Firewall y reglas de tráfico.
- Máquinas virtuales.

## 🌐 Topología de red

El proyecto utiliza tres redes principales:

| Red | Dirección | Función |
|---|---|---|
| Red 10 | `192.168.10.0/24` | Red de conexión entre los pfSense y acceso remoto |
| Red 11 | `192.168.11.0/24` | Red LAN del pfSense11 |
| Red 12 | `192.168.12.0/24` | Red LAN del pfSense12 |

### Direccionamiento de los equipos

| Equipo | Dirección IP |
|---|---|
| pfSense11 (interfaz LAN) | `192.168.11.1` |
| pfSense12 (interfaz LAN) | `192.168.12.1` |
| pfSense11 (interfaz de conexión) | `192.168.10.11` |
| pfSense12 (interfaz de conexión) | `192.168.10.12` |
| Equipo cliente de la red 11 | `192.168.11.31` |
| Equipo cliente de la red 12 | `192.168.12.101` |
| Equipo externo de prueba | `192.168.10.35` |

### Redes virtuales de las VPN

| Conexión | Red del túnel |
|---|---|
| VPN Site-to-Site | `172.16.0.0/24` |
| VPN de acceso remoto | `172.16.1.0/24` |

## 🔐 1. Creación de certificados digitales

Antes de configurar OpenVPN, se preparan los certificados necesarios para establecer conexiones seguras.

En pfSense11 se crea una autoridad certificadora (CA) interna, que se utiliza para firmar los certificados del proyecto.

A continuación, se crean los certificados correspondientes:

- **ServerCert:** certificado del servidor OpenVPN de pfSense11. Inicialmente se había creado con el nombre `CAServer`, pero posteriormente se cambió a `ServerCert`.
- **pfSense12Cert:** certificado utilizado para identificar el segundo pfSense en la conexión Site-to-Site.
- **Certificado del usuario:** certificado creado para el usuario que se conecta mediante la VPN de acceso remoto.

También se exportan los certificados y las claves necesarias para importarlos en el segundo pfSense y en el equipo cliente.

## 🔗 2. Configuración de la VPN Site-to-Site

El objetivo de esta conexión es comunicar las redes `192.168.11.0/24` y `192.168.12.0/24`, permitiendo que los equipos de ambas redes puedan intercambiar tráfico.

### 2.1. Configuración del servidor OpenVPN en pfSense11

En pfSense11 se configura un servidor OpenVPN con los siguientes parámetros:

| Parámetro | Configuración |
|---|---|
| Descripción | Servidor OpenVPN |
| Modo | Peer to Peer (SSL/TLS) |
| Modo de dispositivo | TUN - Layer 3 Tunnel Mode |
| Protocolo | UDP on IPv4 only |
| Interfaz | WAN |
| Puerto local | `1194` |
| Red del túnel | `172.16.0.0/24` |
| Red local IPv4 | `192.168.11.0/24` |
| Red remota IPv4 | `192.168.12.0/24` |
| Autoridad certificadora | CA creada para el proyecto |
| Certificado del servidor | `ServerCert` |

Se habilita el uso de una clave TLS compartida para proteger el canal de control de OpenVPN.

La clave se genera en pfSense11, se copia y se guarda en un fichero de texto para utilizarla posteriormente en pfSense12.

### 2.2. Configuración del segundo pfSense

En pfSense12 se importa la autoridad certificadora y los certificados exportados desde pfSense11.

Después, se configura un cliente OpenVPN para conectarlo con el servidor de pfSense11.

Los parámetros principales son:

| Parámetro | Configuración |
|---|---|
| Modo | Peer to Peer (SSL/TLS) |
| Modo de dispositivo | TUN - Layer 3 Tunnel Mode |
| Protocolo | UDP on IPv4 only |
| Interfaz | WAN |
| Servidor remoto | `192.168.10.11` |
| Puerto del servidor | `1194` |
| Red del túnel | `172.16.0.0/24` |
| Red remota IPv4 | `192.168.11.0/24` |
| Certificado | `pfSense12Cert` |
| Clave TLS | La misma clave compartida configurada en pfSense11 |

Se desactiva la generación automática de la clave TLS para introducir la clave compartida anteriormente guardada.

Una vez configurado, se guarda y se aplica la configuración.

### 2.3. Configuración del firewall para Site-to-Site

Se crean las reglas necesarias para permitir el establecimiento del túnel y el tráfico entre ambas redes.

En pfSense11 se configura una regla WAN que permite las conexiones UDP entrantes por el puerto `1194`.

También se crea una regla en la pestaña OpenVPN para permitir el tráfico desde la red `192.168.12.0/24` hacia las subredes LAN.

En pfSense12 se configura la regla correspondiente para permitir el tráfico desde `192.168.11.0/24` hacia sus subredes LAN.

### 2.4. Comprobación de la conexión

Una vez configurados ambos pfSense, se comprueba que el túnel está establecido y que existe comunicación entre las redes.

Se realizan las siguientes pruebas:

- Desde la red 11 hacia la red 12: ping a `192.168.12.101`.
- Desde la red 12 hacia la red 11: ping a `192.168.11.31`.

Los resultados muestran respuestas correctas en ambas direcciones, confirmando la comunicación entre las dos redes.

## 🌍 3. Configuración de la VPN de acceso remoto

Una vez establecida la conexión Site-to-Site, se configura un segundo servidor OpenVPN en pfSense11 para permitir que usuarios externos puedan conectarse a la red local.

### 3.1. Creación del grupo y del usuario

En pfSense11 se crea un grupo llamado:

`OpenVPN-Users`

Se configura con ámbito local (*Local*).

Posteriormente, se crea un usuario llamado `fran`, que se añade al grupo `OpenVPN-Users`.

Durante la creación del usuario se marca la opción para generar un certificado digital propio.

El certificado creado para este usuario se utiliza en la autenticación de la conexión remota.

### 3.2. Configuración del servidor de acceso remoto

Se añade un segundo servidor OpenVPN en pfSense11 con los siguientes parámetros:

| Parámetro | Configuración |
|---|---|
| Descripción | servervpn |
| Modo | Remote Access (SSL/TLS + User Auth) |
| Backend de autenticación | Local Database |
| Modo de dispositivo | TUN - Layer 3 Tunnel Mode |
| Protocolo | UDP on IPv4 only |
| Interfaz | WAN |
| Puerto local | `1195` |
| Red del túnel | `172.16.1.0/24` |
| Red local IPv4 | `192.168.11.0/24` |
| Certificado del servidor | `ServerCert` |

Se utiliza la autoridad certificadora del proyecto y se mantiene la configuración de clave TLS generada automáticamente para este servidor.

Con esta configuración, los usuarios remotos pueden conectarse a la red `192.168.11.0/24`.

### 3.3. Reglas del firewall para acceso remoto

Se configura una regla en WAN para permitir las conexiones UDP entrantes al puerto `1195`.

También se crea una regla en la pestaña OpenVPN que permite el tráfico de los clientes VPN hacia las subredes LAN.

De esta forma, los usuarios que establezcan correctamente la conexión podrán acceder a los recursos de la red local permitida.

## 💻 4. Exportación y configuración del cliente OpenVPN

Para facilitar la conexión de los usuarios remotos se instala el paquete **OpenVPN Client Export** desde el Package Manager de pfSense.

Una vez instalado:

1. Se accede a `VPN > OpenVPN > Client Export`.
2. Se localiza el usuario `fran`.
3. Se selecciona la configuración correspondiente al cliente Windows.
4. Se descarga el archivo de configuración del cliente.
5. Se descomprime el archivo descargado.
6. Se trasladan los ficheros necesarios al equipo cliente.
7. Se instala OpenVPN en Windows.
8. Se importan los certificados y la configuración exportada.
9. Se inicia la conexión desde el cliente OpenVPN.

De esta manera, el usuario puede establecer una conexión remota con pfSense11 utilizando sus credenciales y certificados.

## 🧪 5. Pruebas de funcionamiento

Se realizan diferentes pruebas para verificar el funcionamiento de las dos VPN.

### Prueba 1: comunicación entre las redes 11 y 12

Se comprueba la comunicación entre los equipos de ambas redes mediante ping.

- Red 11 → Red 12: `192.168.12.101`.
- Red 12 → Red 11: `192.168.11.31`.

**Resultado:** comunicación correcta en ambos sentidos.

### Prueba 2: conexión de un usuario remoto

Se utiliza un equipo externo situado en la red `192.168.10.0/24`, concretamente con la dirección `192.168.10.35`.

Desde este equipo se configura e inicia el cliente OpenVPN utilizando los ficheros exportados desde pfSense11.

Una vez establecida la conexión, se realiza un ping al equipo `192.168.11.31`.

**Resultado:** se obtiene respuesta del equipo de la red 11, verificando que el acceso remoto funciona correctamente.

## ✅ 6. Resultado final

Al finalizar el proyecto se consigue:

- Crear una autoridad certificadora y los certificados necesarios.
- Configurar un servidor OpenVPN Site-to-Site en pfSense11.
- Configurar pfSense12 como cliente OpenVPN Site-to-Site.
- Establecer comunicación entre las redes `192.168.11.0/24` y `192.168.12.0/24`.
- Configurar un grupo y un usuario para el acceso remoto.
- Crear un segundo servidor OpenVPN para usuarios remotos.
- Configurar las reglas de firewall necesarias.
- Exportar la configuración del cliente mediante OpenVPN Client Export.
- Conectar un equipo externo a la red local del pfSense11.

Las pruebas realizadas confirman el funcionamiento de la conexión Site-to-Site y del acceso remoto.

## 📁 Contenido del repositorio

Este repositorio recoge la documentación y el procedimiento seguido para realizar la configuración de las VPN mediante pfSense y OpenVPN.

Se incluyen las configuraciones realizadas, el direccionamiento utilizado y las pruebas de conectividad que permiten comprobar el resultado del proyecto.

## 👤 Autor

Proyecto realizado como práctica de configuración de redes, seguridad perimetral y conexiones VPN utilizando pfSense y OpenVPN.
