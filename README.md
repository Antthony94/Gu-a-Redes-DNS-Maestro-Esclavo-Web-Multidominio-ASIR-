# 🧠 Guía Explicada Paso a Paso: DNS Maestro/Esclavo + Web Multidominio (ASIR)

> Esta guía **no es solo para copiar y pegar**. Está pensada para que **sepas en todo momento dónde estás, qué estás haciendo, por qué lo haces y cómo continuar**, exactamente como te lo pedirían en un examen práctico de ASIR.

---

## 📌 0. Contexto General: ¿Qué estamos montando?

Vamos a construir **una infraestructura completa típica de examen**:

* Un **servidor DNS Maestro** que gestiona varios dominios.
* Un **servidor DNS Esclavo** que replica automáticamente las zonas.
* Un **servidor web Apache** que aloja **varias páginas web** (una por dominio).

Todo funcionando **con resolución DNS real**, sin `/etc/hosts`.

---

## 🧪 1. Escenario del Examen (Dónde estamos)

### 📡 Red

* Red interna: `172.16.102.0/24`

### 🖥️ Máquinas

| Máquina       | Rol                   | IP           |
| ------------- | --------------------- | ------------ |
| Ubuntu Server | DNS Maestro + Apache  | 172.16.102.5 |
| Ubuntu Redes  | DNS Esclavo + Cliente | 172.16.102.1 |

### 🌍 Dominios que vamos a gestionar

* `anthony.ies`
* `tienda.ies`
* `blog.ies`

👉 **Objetivo final**: que desde el cliente pueda entrar a los 3 dominios por navegador y que el DNS funcione incluso si el maestro cae.

---

## 🛠️ 2. Preparación Inicial (Qué hacemos primero y por qué)

Antes de configurar nada, **ambas máquinas deben tener las herramientas necesarias**.

### 2.1 Instalación de paquetes

📍 Estamos en: **Servidor y Cliente**

```bash
sudo apt update
sudo apt install bind9 dnsutils apache2
```

🔎 **Por qué**:

* `bind9`: servidor DNS
* `dnsutils`: herramientas de prueba (`dig`, `nslookup`)
* `apache2`: servidor web (solo se usará en el Server, pero no molesta en el cliente)

---

### 2.2 Error CRÍTICO de laboratorio: SERVFAIL

📍 Seguimos en **ambas máquinas**

En entornos educativos, **DNSSEC provoca errores**. Si no lo desactivas, el DNS falla aunque esté bien configurado.

Editar:

```bash
sudo nano /etc/bind/named.conf.options
```

```conf
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
    };

    dnssec-validation no;

    listen-on-v6 { any; };
};
```

✅ **Qué hemos hecho**: garantizar que el DNS no se rompe por validaciones externas.

---

## 🦁 3. DNS Maestro (Servidor 172.16.102.5)

📍 Ahora estamos **solo en el servidor maestro**.

Aquí es donde **se crean y gestionan las zonas DNS**. El esclavo solo copiará.

---

### 3.1 Declarar las zonas (Decirle a Bind qué dominios existen)

Archivo:

```bash
sudo nano /etc/bind/named.conf.local
```

Aquí declaramos:

* 3 zonas directas
* 1 zona inversa
* Permitimos que el esclavo copie los datos

```conf
zone "anthony.ies" {
    type master;
    file "/etc/bind/zones/db.anthony.ies";
    allow-transfer { 172.16.102.1; };
    also-notify { 172.16.102.1; };
};

zone "tienda.ies" {
    type master;
    file "/etc/bind/zones/db.tienda.ies";
    allow-transfer { 172.16.102.1; };
    also-notify { 172.16.102.1; };
};

zone "blog.ies" {
    type master;
    file "/etc/bind/zones/db.blog.ies";
    allow-transfer { 172.16.102.1; };
    also-notify { 172.16.102.1; };
};

zone "102.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.172";
    allow-transfer { 172.16.102.1; };
    also-notify { 172.16.102.1; };
};
```

🧠 **Qué está pasando**:

* `type master`: este servidor manda
* `allow-transfer`: permitimos réplica
* `also-notify`: avisamos al esclavo de cambios

---

### 3.2 Crear los archivos de zona (Los datos reales del DNS)

📍 Seguimos en el **Servidor Maestro**

Primero creamos la carpeta:

```bash
sudo mkdir -p /etc/bind/zones
```

---

#### 3.2.1 Zona anthony.ies

```bash
sudo nano /etc/bind/zones/db.anthony.ies
```

```dns
$TTL 604800
@ IN SOA anthony.ies. root.anthony.ies. (
    5 604800 86400 2419200 604800 )

@ IN NS ns.anthony.ies.
@ IN A 172.16.102.5

ns IN A 172.16.102.5
client IN A 172.16.102.1
router IN A 172.16.102.100

www IN CNAME ns
```

🧠 **Qué hemos definido**:

* Quién es el DNS
* Qué IP tiene cada nombre
* Alias `www`

---

#### 3.2.2 Zona tienda.ies

```bash
sudo nano /etc/bind/zones/db.tienda.ies
```

```dns
$TTL 604800
@ IN SOA tienda.ies. root.tienda.ies. (
    1 604800 86400 2419200 604800 )

@ IN NS ns.tienda.ies.
@ IN A 172.16.102.5

ns IN A 172.16.102.5
www IN CNAME ns
```

---

#### 3.2.3 Zona blog.ies

```bash
sudo nano /etc/bind/zones/db.blog.ies
```

```dns
$TTL 604800
@ IN SOA blog.ies. root.blog.ies. (
    1 604800 86400 2419200 604800 )

@ IN NS ns.blog.ies.
@ IN A 172.16.102.5

ns IN A 172.16.102.5
www IN CNAME ns
```

---

#### 3.2.4 Zona inversa

```bash
sudo nano /etc/bind/zones/db.172
```

```dns
$TTL 604800
@ IN SOA ns.anthony.ies. root.anthony.ies. (
    3 604800 86400 2419200 604800 )

@ IN NS ns.anthony.ies.
5 IN PTR ns.anthony.ies.
1 IN PTR client.anthony.ies.
```

---

### 3.3 Aplicar cambios

```bash
sudo systemctl restart bind9
```

✅ El maestro ya funciona.

---

## 🐵 4. DNS Esclavo (172.16.102.1)

📍 Ahora estamos en el **servidor esclavo**.

Este **NO crea zonas**, solo las copia.

```bash
sudo nano /etc/bind/named.conf.local
```

```conf
zone "anthony.ies" {
    type secondary;
    file "db.anthony.ies";
    masters { 172.16.102.5; };
};

zone "tienda.ies" {
    type secondary;
    file "db.tienda.ies";
    masters { 172.16.102.5; };
};

zone "blog.ies" {
    type secondary;
    file "db.blog.ies";
    masters { 172.16.102.5; };
};

zone "102.16.172.in-addr.arpa" {
    type secondary;
    file "db.172";
    masters { 172.16.102.5; };
};
```

```bash
sudo systemctl restart bind9
ls -l /var/cache/bind/
```

✅ Si ves los archivos, la transferencia funciona.

---

## 🌐 5. Apache Multidominio (Servidor)

📍 Volvemos al **Servidor Maestro**.

### 5.1 Crear carpetas web

```bash
sudo mkdir -p /var/www/anthony /var/www/tienda /var/www/blog
```

Cada dominio tendrá **su propia web**.

---

### 5.2 Virtual Hosts (Decirle a Apache qué web servir)

📍 `/etc/apache2/sites-available/`

Ejemplo:

```apache
<VirtualHost *:80>
    ServerName www.anthony.ies
    ServerAlias anthony.ies
    DocumentRoot /var/www/anthony
</VirtualHost>
```

(Repetir para tienda y blog)

---

### 5.3 Activación

```bash
sudo a2dissite 000-default.conf
sudo a2ensite anthony.conf tienda.conf blog.conf
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/
sudo systemctl reload apache2
```

---

## ✅ 6. Comprobación Final (Esto es lo que mira el profe)

```bash
nslookup anthony.ies
dig @172.16.102.1 anthony.ies axfr
```

🌍 Navegador:

* [http://www.anthony.ies](http://www.anthony.ies)
* [http://www.tienda.ies](http://www.tienda.ies)
* [http://www.blog.ies](http://www.blog.ies)

---

🎓 **Si entiendes cada paso de este documento, estás preparado para el examen.**
