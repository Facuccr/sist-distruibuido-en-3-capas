# Arquitectura Distribuida de 3 Niveles en Máquinas Virtuales Linux

## Descripción del Proyecto

Este proyecto consiste en la implementación de una aplicación de 3 capas operando en un entorno de red aislada. El objetivo principal es lograr la separación física de los servicios utilizando máquinas virtuales independientes.

La infraestructura se basa en:

- **Capa de Datos (VM 1):** Servidor de Base de Datos.
- **Capa Lógica (VM 2):** Servidor Backend (API).
- **Capa de Presentación (VM 3):** Servidor Web Frontend.

La comunicación inter-VM se gestiona mediante IPs estáticas dentro de un segmento de red privado, asegurando el acceso remoto a través de túneles de reenvío de puertos.

---

## Tecnologías Utilizadas

- **Virtualización e Infraestructura:** Oracle VirtualBox, Ubuntu Server 24.04 LTS.
- **Modo de Red:** Red NAT (NAT Network).
- **Base de Datos:** MariaDB Server.
- **Backend:** Node.js, Express, mysql2, CORS.
- **Frontend:** Apache2, HTML5 puro, JavaScript (Fetch API).

---

## Infraestructura de Red Implementada

| Máquina  | Rol                   | IP Estática Interna | Puerto del Servicio |
| :------- | :-------------------- | :------------------ | :------------------ |
| **VM 1** | Base de Datos MariaDB | `10.0.2.15`         | `3306`              |
| **VM 2** | Backend API Node.js   | `10.0.2.16`         | `3454`              |
| **VM 3** | Servidor Web Apache   | `10.0.2.17`         | `8080`              |

---

## Guía de Implementación

### Paso 1: Instalación y Optimización del Sistema Base

Se utilizó la imagen oficial `ubuntu-24.04.4-live-server-amd64.iso`. Durante la instalación se habilitó OpenSSH Server.

Para optimizar los tiempos de arranque en las instancias virtualizadas, se procedió a deshabilitar los servicios de inicialización en la nube ejecutando la siguiente secuencia:

```bash
sudo touch /etc/cloud/cloud-init.disabled
sudo systemctl stop cloud-init
sudo systemctl disable cloud-init
sudo systemctl mask cloud-init
sudo systemctl disable systemd-networkd-wait-online.service
sudo systemctl mask systemd-networkd-wait-online.service
```

### Paso 2: Configuración de Enrutamiento (Red NAT)

Para permitir que las máquinas interactúen entre sí utilizando IPs estáticas mientras mantienen salida a internet y acceso desde el anfitrión, se implementó una configuración de "Red NAT" a nivel global en VirtualBox.

- **Nombre de la red:** `NatNetwork`
- **Prefijo IPv4:** `10.0.2.0/24`

Las tres máquinas virtuales fueron vinculadas a este único adaptador de red, eliminando el aislamiento del modo NAT simple.

### Paso 3: Asignación de Direccionamiento Estático

Se configuró el administrador de red Netplan en cada servidor editando el archivo `/etc/netplan/50-cloud-init.yaml`. Se aplicó el siguiente bloque, modificando la dirección IP según el rol de la máquina (`10.0.2.15`, `10.0.2.16` o `10.0.2.17`):

```yaml
network:
  version: 2
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
```

Se validó la configuración ejecutando `sudo netplan apply`.

### Paso 4: Configuración de la Capa de Datos (VM 1)

Se instaló el motor relacional y se ejecutó la configuración de seguridad:

```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

Se habilitó la escucha en todas las interfaces modificando el archivo `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```ini
bind-address = 0.0.0.0
```

Se generó el esquema de la base de datos, la tabla de registro y el usuario de acceso remoto:

```sql
CREATE DATABASE sistema_escolar;
USE sistema_escolar;

CREATE TABLE alumnos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    apellidos VARCHAR(100) NOT NULL,
    nombres VARCHAR(100) NOT NULL,
    dni VARCHAR(20) UNIQUE
);

CREATE USER 'usuario-consulta'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON sistema_escolar.* TO 'usuario-consulta'@'%';
FLUSH PRIVILEGES;
```

### Paso 5: Configuración de la Capa Lógica (VM 2)

Se preparó el entorno de ejecución instalando Node.js e inicializando el proyecto:

```bash
sudo apt install nodejs npm -y
mkdir api_backend && cd api_backend
npm init -y
npm install express mysql2 cors
```

Se implementó el archivo `server.js` conectando directamente a la IP de la VM 1 y proveyendo los endpoints requeridos:

```javascript
const express = require("express");
const mysql = require("mysql2");
const cors = require("cors");
const app = express();

app.use(express.json());
app.use(cors());

const db = mysql.createConnection({
  host: "10.0.2.15",
  user: "usuario-consulta",
  password: "admin123",
  database: "sistema_escolar",
  port: 3306,
});

app.post("/grabaAlumnos", (req, res) => {
  const { nombres, apellidos, dni } = req.body;
  const checkQuery = "SELECT count(*) AS cantidad FROM alumnos WHERE dni = ?";

  db.query(checkQuery, [dni], (err, results) => {
    if (err) return res.status(500).json({ error: err.message });

    if (results[0].cantidad > 0) {
      return res.json({ status: 0, message: "DNI Duplicado" });
    } else {
      const insertQuery =
        "INSERT INTO alumnos (nombres, apellidos, dni) VALUES (?, ?, ?)";
      db.query(insertQuery, [nombres, apellidos, dni], (err) => {
        if (err) return res.status(500).json({ error: err.message });
        return res.json({ status: 1, message: "Exito" });
      });
    }
  });
});

app.get("/consultarAlumnos", (req, res) => {
  const query = "SELECT * FROM alumnos ORDER BY apellidos ASC, nombres ASC";
  db.query(query, (err, results) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(results);
  });
});

app.listen(3454, () => {
  console.log("API Activa");
});
```

### Paso 6: Configuración de la Capa de Presentación (VM 3)

Se instaló el servidor web y se modificó su puerto de escucha predeterminado al 8080:

```bash
sudo apt install apache2 -y
sudo nano /etc/apache2/ports.conf
```

_(Se estableció `Listen 8080` y se actualizó el VirtualHost correspondiente)._

En el directorio `/var/www/html/Sistema` se creó el archivo `index.html` integrando Fetch API para consumir la API alojada en la VM 2:

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <title>Registro de Alumnos</title>
  </head>
  <body>
    <h1>Cargar Alumno</h1>
    <form id="alumnoForm">
      <input type="text" id="nombres" placeholder="Nombres" required />
      <input type="text" id="apellidos" placeholder="Apellidos" required />
      <input type="text" id="dni" placeholder="DNI" required />
      <button type="submit">Guardar</button>
    </form>

    <h2>Listado de Alumnos</h2>
    <button onclick="cargarTabla()">Consultar</button>
    <table border="1" id="tablaAlumnos">
      <thead>
        <tr>
          <th>Apellidos</th>
          <th>Nombres</th>
          <th>DNI</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>

    <script>
      const apiUrl = "http://localhost:3454";

      document
        .getElementById("alumnoForm")
        .addEventListener("submit", function (e) {
          e.preventDefault();
          const data = {
            nombres: document.getElementById("nombres").value,
            apellidos: document.getElementById("apellidos").value,
            dni: document.getElementById("dni").value,
          };

          fetch(`${apiUrl}/grabaAlumnos`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(data),
          })
            .then((res) => res.json())
            .then((res) => {
              if (res.status === 1) {
                alert("Guardado");
                cargarTabla();
              } else {
                alert("Error: " + res.message);
              }
            });
        });

      function cargarTabla() {
        fetch(`${apiUrl}/consultarAlumnos`)
          .then((res) => res.json())
          .then((data) => {
            const tbody = document.querySelector("#tablaAlumnos tbody");
            tbody.innerHTML = "";
            data.forEach((alumno) => {
              tbody.innerHTML += `<tr>
                        <td>${alumno.apellidos}</td>
                        <td>${alumno.nombres}</td>
                        <td>${alumno.dni}</td>
                    </tr>`;
            });
          });
      }
    </script>
  </body>
</html>
```

### Paso 7: Reglas de Port Forwarding

Para administrar los servidores sin interfaz gráfica y acceder a la aplicación desde el navegador del sistema operativo anfitrión, se establecieron las siguientes reglas en el administrador de Red NAT:

| Regla           | Protocolo | IP Host     | Puerto Host | IP Invitado | Puerto Invitado |
| :-------------- | :-------- | :---------- | :---------- | :---------- | :-------------- |
| **SSH_DB**      | TCP       | `127.0.0.1` | `2221`      | `10.0.2.15` | `22`            |
| **SSH_BACK**    | TCP       | `127.0.0.1` | `2222`      | `10.0.2.16` | `22`            |
| **SSH_FRONT**   | TCP       | `127.0.0.1` | `2223`      | `10.0.2.17` | `22`            |
| **API_SERVICE** | TCP       | `127.0.0.1` | `3454`      | `10.0.2.16` | `3454`          |
| **WEB_SISTEMA** | TCP       | `127.0.0.1` | `8080`      | `10.0.2.17` | `8080`          |

---

## Resultados y Validación

El despliegue fue completado exitosamente. El backend asegura la integridad de los datos evitando duplicados por DNI, y el sistema en su conjunto valida el modelo de 3 capas operando bajo una arquitectura distribuida y centralizada a través de un ruteo NAT interno.

**Acceso al sistema:** `http://localhost:8080/Sistema/`
<img width="1275" height="798" alt="{B13D08E7-2C07-480F-BA57-CE2EC371D00A}" src="https://github.com/user-attachments/assets/151863e0-97ab-4e7f-b854-1b8af24544dc" />

<img width="869" height="721" alt="{8B1FEFC4-74B5-4096-A579-1BD72774DF99}" src="https://github.com/user-attachments/assets/74ee3741-e5d7-4710-8f9b-1bf93a354829" />
