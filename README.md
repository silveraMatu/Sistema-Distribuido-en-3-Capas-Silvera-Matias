# Sistema-Distribuido-en-3-Capas-Silvera-Matias
# Trabajo Práctico - Sistema Distribuido en 3 Capas (VirtualBox & Ubuntu Server)

Este proyecto consiste en el diseño e implementación de una arquitectura distribuida real de tres capas utilizando máquinas virtuales Linux sobre Oracle VirtualBox. El principal objetivo pedagógico es la separación absoluta de responsabilidades en la infraestructura, configurando el almacenamiento de datos persistente, la lógica de negocio a través de una API REST y la interfaz gráfica de usuario en entornos aislados e intercomunicados de forma estática.

---

## 📐 Arquitectura de Infraestructura de Red

Se implementó un direccionamiento estático bajo una **Red NAT** global (`10.0.2.0/24`) para asegurar la intercomunicación directa entre los nodos de servicios y habilitar la salida hacia internet para la instalación de dependencias

| Nodo Virtual | Rol del Servidor | IP Estática Interna | Puerto del Servicio | Puerto SSH (Host) |
| :--- | :--- | :--- | :--- | :--- |
| **VM1-DB** | Base de Datos Relacional | `10.0.2.15` | `3306` (MariaDB) | `2221` |
| **VM2-BACKEND** | Lógica de Negocio / API REST | `10.0.2.16` | `3454` (Node.js) | `2222` |
| **VM3-FRONTEND** | Servidor Web / Interfaz | `10.0.2.17` | `8080` (Apache2) | `2223` |

---

## 🛠️ Tecnologías y Herramientas Utilizadas

**Hipervisor & Base:** Oracle VirtualBox & Ubuntu Server 24.04 LTS

**Capa de Almacenamiento (VM1):** MariaDB Server & SQL

**Capa de Negocio (VM2):** Node.js, Express, MySQL2 Drivers y Middelware CORS

**Capa de Presentación (VM3):** Servidor HTTP Apache2, HTML5 semántico, JavaScript nativo (Fetch API)

**Desarrollo & Control:** Visual Studio Code (en máquina Anfitrión/Host), SSH Client, SCP

---

## Paso a Paso: Implementación Completa

### Paso 1: Configuración de la Red NAT en el Anfitrión (Host)
1. Abre VirtualBox en tu sistema anfitrión y dirígete a **Archivo** > **Herramientas** > **Red** (o Preferencias > Red dependiendo de la versión)
2. Crea una nueva **Red NAT** (por defecto `NatNetwork`)
3. Configura el CIDR de red a exactamente `10.0.2.0/24` y verifica que la casilla **Soportar DHCP** esté activada (necesario solo para la fase inicial de aprovisionamiento de paquetes)

### Paso 2: Creación y Optimización de la VM Base (`ubuntu-base`)
1. Crea una nueva máquina virtual con los siguientes recursos asignados
   * **RAM:** 1536 MB
   * **CPU:** 2 Núcleos
   * **Disco:** 25 GB de almacenamiento dinámico
   * **Red:** Adaptador 1 conectado a **Red NAT** utilizando la red `NatNetwork` creada en el paso anterior.
     
2. Inicia la VM, monta la ISO oficial e instala Ubuntu Server.
3. **Resolución de DNS (Fase de Instalación):** Si durante el despliegue inicial notas errores de tipo *Fallo temporal al resolver*, ingresa temporalmente los DNS de Google en la máquina mediante:
   ```bash
   echo -e "nameserver 8.8.8.8\nnameserver 8.8.4.4" | sudo tee /etc/resolv.conf
   ```
4. Actualiza la lista de paquetes locales e instala explícitamente el servidor de accesos remotos:
   ```bash
   sudo apt update && sudo apt install openssh-server -y
   ```

5. Desactivación de Cloud-Init: Ejecuta el bloque de comandos para remover las validaciones de red iniciales del Kernel y acelerar el tiempo de arranque del sistema operativo en cada nodo:
   ```bash
   sudo touch /etc/cloud/cloud-init.disabled
   sudo systemctl stop cloud-init
   sudo systemctl disable cloud-init
   sudo systemctl mask cloud-init
   sudo systemctl disable cloud-config cloud-final cloud-init-local
   sudo systemctl disable systemd-networkd-wait-online.service
   sudo systemctl mask systemd-networkd-wait-online.service
   ```
6. Apagamos la máquina virtual base:
   ```bash
   sudo poweroff
   ```

### Paso 3: Clonación y Direccionamiento Estático (Netplan)

Realiza tres clonaciones desde ubuntu-base nombrando las instancias como VM1-DB, VM2-BACKEND y VM3-FRONTEND.  CRÍTICO: Asegúrate de marcar de forma obligatoria la opción "Generar nuevas direcciones MAC para todas las tarjetas de red" para mitigar colisiones y solapamiento de identificadores físicos de red en el hipervisor.  Enciende las tres máquinas virtuales. Accede a cada una de forma independiente mediante la interfaz del hipervisor y modifica la configuración de Netplan con el editor de texto:  

```Bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Estructura del archivo (Ejemplo base correspondiente a la VM1 con IP `10.0.2.15/24`):   
```YAML
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```
Ajustar en VM2 la línea addresses a `- 10.0.2.16/24`.  
Ajustar en VM3 la línea addresses a `- 10.0.2.17/24`.  

En cada terminal, aplica los cambios estructurales:  

```Bash
sudo netplan apply
```

### Paso 4: Enrutamiento de Puertos (Port Forwarding Global)

Apaga o minimiza las máquinas y regresa al panel de VirtualBox. En la configuración global de tu Red NAT, añade de forma estricta las siguientes políticas de reenvío para habilitar la administración distribuida desde tu terminal del Host:  
| Regla | Protocolo | IP Anfitrión | Puerto Anfitrión | IP Invitado | Puerto Invitado | 
| :--- | :--- | :--- | :--- | :--- | :--- |
| BACKEND | TCP | 127.0.0.1 | 3454 | 10.0.2.16 | 3454 |
| HTTP_FRONT | TCP| 127.0.0.1 | 8080 | 10.0.2.17 | 8080 |
| SSH_VM1 | TCP | 127.0.0.1 | 2221 | 10.0.2.15 | 22 |
| SSH_VM2 | TCP | 127.0.0.1 | 2222 | 10.0.2.16 | 22
| SSH_VM3 | TCP | 127.0.0.1 | 2223 | 10.0.2.17 | 22 |

A partir de este punto, todas las tareas se realizan de forma ágil desde el Host usando SSH:
```bash
ssh matu@localhost -p 2221  # Entrada a la VM1
```
---

**Detalle de Implementación: VM1 - Servidor de Base de Datos**

***Tecnologías Aplicadas***

| Motor de Base de Datos Relacional | Lenguaje|
| :--- | :--- |
| MariaDB 10.x | SQL estructurado estándar |

**Procedimiento**

Accede remotamente al nodo: 

```bash
ssh matu@localhost -p 2221.  
```

Instala e inicializa el daemon del motor relacional:  

```Bash
sudo apt update && sudo apt install mariadb-server -y
```
Ingresa a la interfaz de administración del motor:  
```Bash
sudo mysql -u root
```
Define el esquema lógico, la tabla operativa y los esquemas de credenciales de conexión remota:  

```SQL
CREATE DATABASE escuela;
USE escuela;

CREATE TABLE alumnos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    apellidos VARCHAR(100),
    nombres VARCHAR(100),
    dni VARCHAR(20) UNIQUE
);

-- Creación del usuario remoto restringido
CREATE USER 'usuario_consulta'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON escuela.* TO 'usuario_consulta'@'%';
FLUSH PRIVILEGES;
EXIT;
```
Configuración del Socket de Escucha Externa: Por defecto, MariaDB deniega conexiones que no provengan del lazo local loopback. Edita las directivas del server:  

```Bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Ubica la variable de red `bind-address` y cambia su valor restrictivo: 

```Plaintext
bind-address = 0.0.0.0
```
Reinicia el servicio para asentar las directivas en caliente:  

```Bash
sudo systemctl restart mariadb
```
---

**Detalle de Implementación: VM2 - Servidor Backend**

***Tecnologías Aplicadas***

| Entorno de Ejecución:  | Servidor HTTP & Enrutamiento  | Driver de conexión relacional | Seguridad de Cabeceras |
|:---|:---|:---|:---|
| Node.js nativo.| Express Framework.|  mysql2 driver.|  Cors Engine (para desestimar bloqueos de origen cruzado).|  

**Procedimiento**

En tu máquina host, crea una carpeta de desarrollo y tu archivo de lógica usando tu editor VS Code favorito:  

```Bash
mkdir backend && cd backend && touch server.js
```
Escribe los endpoints requeridos por la lógica en server.js asegurando apuntar al host de la base datos `10.0.2.15`:

```JavaScript
const express = require('express');
const mysql = require('mysql2');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

// Conector persistente apuntando a la VM1
const db = mysql.createConnection({
    host: '10.0.2.15',
    user: 'usuario_consulta',
    password: '1234',
    database: 'escuela'
});

// POST: Registrar Alumno con validación lógica de unicidad de DNI
app.post('/grabaAlumnos', (req, res) => {
    const { apellidos, nombres, dni } = req.body;

    db.query('SELECT * FROM alumnos WHERE dni = ?', [dni], (err, results) => {
        if (err) return res.status(500).send(err);
        if (results.length > 0) return res.send("0"); // Error: DNI duplicado

        db.query('INSERT INTO alumnos (apellidos, nombres, dni) VALUES (?, ?, ?)', 
        [apellidos, nombres, dni], (err, result) => {
            if (err) return res.status(500).send(err);
            res.send("1"); // Inserción exitosa
        });
    });
});

// GET: Retornar listado clasificado por criterios alfabéticos
app.get('/consultarAlumnos', (req, res) => {
    db.query('SELECT * FROM alumnos ORDER BY apellidos ASC, nombres ASC', (err, results) => {
        if (err) return res.status(500).send(err);
        res.json(results);
    });
});

app.listen(3454, () => {
    console.log('Backend corriendo en el puerto 3454');
});
```

Conéctate a la VM2 mediante SSH para preparar el entorno e instalar dependencias:  
```Bash
ssh matu@localhost -p 2222
sudo apt update && sudo apt install nodejs npm -y
mkdir ~/app && cd ~/app
npm init -y
npm install express mysql2 cors
```
Abre otra terminal local en tu máquina host y despacha el archivo desarrollado por canal SCP seguro hacia el directorio de la VM:  
```Bash
scp -P 2222 server.js matu@localhost:~/app/
```
Regresa a la terminal SSH de tu VM2 y ejecuta el proceso:  
```Bash
node ~/app/server.js
```
---

**Detalle de Implementación: VM3 - Servidor Frontend**
***Tecnologías Aplicadas***
|Servidor Web Engine |Maquetación & Control |Integración asíncrona|
|:---|:---|:---|
|Apache2 HTTP Server| HTML5 + Vanilla JavaScript Sintáctico| Fetch API|
 
**Procedimiento**
En tu máquina host, crea una carpeta para la interfaz cliente y añade los archivos base:  
```Bash
mkdir Frontend && cd Frontend && touch index.html app.js
```

`index.html`: Desarrolla el formulario simple estructurando los campos de texto correspondientes:

```HTML
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Sistema de Alumnos - Arquitectura Distribuida</title>
</head>
<body>
    <h1>Registro de Alumnos</h1>
    <form id="formAlumno">
        <input type="text" id="apellidos" placeholder="Apellidos" required>
        <input type="text" id="nombres" placeholder="Nombres" required>
        <input type="text" id="dni" placeholder="DNI" required>
        <button type="submit">Guardar</button>
        <button type="button" id="btnConsultar">Consultar</button>
    </form>
    <div id="resultado" style="margin-top:20px;"></div>
    <script src="app.js"></script>
</body>
</html>
```
`app.js`: Añade la lógica asíncrona. Dado que las peticiones se procesan desde el navegador web de tu host, debes pegarle al Backend a través del puerto mapeado de la máquina física (`localhost:3454`):

```JavaScript
const URL_API = 'http://localhost:3454';

document.getElementById('formAlumno').addEventListener('submit', async (e) => {
    e.preventDefault();
    const apellidos = document.getElementById('apellidos').value;
    const nombres = document.getElementById('nombres').value;
    const dni = document.getElementById('dni').value;

    const response = await fetch(`${URL_API}/grabaAlumnos`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ apellidos, nombres, dni })
    });

    const resText = await response.text();
    if (resText === "1") {
        alert("Alumno guardado con éxito");
        document.getElementById('formAlumno').reset();
    } else {
        alert("Error: El número de DNI ingresado ya existe.");
    }
});

document.getElementById('btnConsultar').addEventListener('click', async () => {
    const response = await fetch(`${URL_API}/consultarAlumnos`);
    const alumnos = await response.json();
    const divResultado = document.getElementById('resultado');
    divResultado.innerHTML = '<h3>Lista de Alumnos Almacenados:</h3>';

    alumnos.forEach(al => {
        divResultado.innerHTML += `<p>${al.apellidos}, ${al.nombres} - DNI: ${al.dni}</p>`;
    });
});
```
Conéctate a la VM3 para aprovisionar Apache2:  
```Bash
ssh matu@localhost -p 2223
sudo apt update && sudo apt install apache2 -y
```
Modifica el puerto de escucha por defecto (80) al puerto requerido (8080) en los archivos del Web Server:  
```Bash
sudo nano /etc/apache2/ports.conf
```
Cambiar: `Listen 80` -> `Listen 8080`

```bash 
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Cambiar: `<VirtualHost *:80>` -> `<VirtualHost *:8080>`

Aplica un reinicio del daemon:  
```Bash
sudo systemctl restart apache2
```
Adjudica permisos del directorio raíz web a tu usuario operativo de la VM para agilizar la transferencia remota de datos:  
```Bash
sudo chown -R $USER:$USER /var/www/html/
mkdir -p /var/www/html/Frontend
```
Desde la terminal local de tu máquina host, exporta recursivamente la carpeta completa mediante SCP:
```Bash
scp -P 2223 -r Frontend matu@localhost:/var/www/html/
```

---

**Verificación Final y Testeo***
Abre tu navegador de preferencia en la computadora anfitriona (física) y dirígete al punto de montaje mapeado de la interfaz de la VM3:  

```Plaintext
http://localhost:8080/Frontend/
```
El sistema debe desplegar el formulario, permitiendo persistir datos dinámicamente sobre MariaDB tras validar las lógicas del Backend en Express.  

#### Capturas de Pantalla del Sistema en Funcionamiento

A continuación, se documenta la evidencia visual del correcto despliegue, inserción y consulta en arquitectura distribuida:

1. Interfaz de Carga del Sistema (Front)

<img width="779" height="305" alt="Sistema funcionando" src="https://github.com/user-attachments/assets/d7bb4a2f-b79d-45dc-b24b-7e66633e313d" />


2. Confirmación de Alumno Guardado Exitosamente
<img width="1233" height="333" alt="Captura desde 2026-05-17 15-51-45" src="https://github.com/user-attachments/assets/87bbc641-46dd-426d-ace1-20d8971b409a" />


3. Consulta y Listado de Datos Persistidos Ordenados
<img width="284" height="159" alt="image" src="https://github.com/user-attachments/assets/a16c1f30-818e-4211-9d13-99315796293b" />
