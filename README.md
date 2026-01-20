# 🧠 Guía Redes: DNS Maestro/Esclavo + Web Multidominio (ASIR)

Guía completa **Para Luis Ruz** como referencia de examen y laboratorio.

---

## 🧪 Escenario del Examen

* **Red:** `172.16.102.0/24`
* **Servidor DNS Maestro + Web (Ubuntu Server):**

  * IP: `172.16.102.5`
  * Hostname: `anthony`
* **Servidor DNS Esclavo / Cliente (Ubuntu Redes):**

  * IP: `172.16.102.1`

### 🌍 Dominios

* `anthony.ies`
* `tienda.ies`
* `blog.ies`

---

## 🛠️ PARTE 1: Configuración Previa (Server y Cliente)

### 1.1 Instalación de Paquetes

En **ambas máquinas**:

```bash
sudo apt update
sudo apt install bind9 dnsutils apache2
```

---

### 1.2 Solución al Error SERVFAIL (CRÍTICO EN EXAMEN)

Desactivar **DNSSEC** para evitar errores de validación.

Editar:

```bash
sudo nano /etc/bind/named.conf.options
```

Contenido:

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

---

## 🦁 PARTE 2: Servidor DNS Maestro (172.16.102.5)

### 2.1 Declaración de Zonas

Archivo:

```bash
sudo nano /etc/bind/named.conf.local
```

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

---

### 2.2 Archivos de Zona

```bash
sudo mkdir -p /etc/bind/zones
```

#### A) db.anthony.ies

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

---

#### B) db.tienda.ies

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

#### C) db.blog.ies

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

#### D) Zona Inversa db.172

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

### 2.3 Reiniciar DNS

```bash
sudo systemctl restart bind9
```

---

## 🐵 PARTE 3: Servidor DNS Esclavo (172.16.102.1)

Archivo:

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

---

## 🌐 PARTE 4: Apache Web Multidominio (Server)

### 4.1 Directorios Web

```bash
sudo mkdir -p /var/www/anthony /var/www/tienda /var/www/blog
```

### 4.2 Virtual Hosts

Archivos en `/etc/apache2/sites-available/`

#### anthony.conf

```apache
<VirtualHost *:80>
    ServerName www.anthony.ies
    ServerAlias anthony.ies
    DocumentRoot /var/www/anthony
</VirtualHost>
```

#### tienda.conf

```apache
<VirtualHost *:80>
    ServerName www.tienda.ies
    ServerAlias tienda.ies
    DocumentRoot /var/www/tienda
</VirtualHost>
```

#### blog.conf

```apache
<VirtualHost *:80>
    ServerName www.blog.ies
    ServerAlias blog.ies
    DocumentRoot /var/www/blog
</VirtualHost>
```

### 4.3 Activación

```bash
sudo a2dissite 000-default.conf
sudo a2ensite anthony.conf tienda.conf blog.conf
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/
sudo systemctl reload apache2
```

---

## 🔥 PARTE 5: Troubleshooting Rápido

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 80/tcp
```

```bash
sudo journalctl -u bind9 -f
tail -f /var/log/apache2/error.log
```

---

## ✅ PARTE 6: Comprobación Final

```bash
nslookup anthony.ies
dig @172.16.102.1 anthony.ies axfr
```

### 🌐 Navegador

* [http://www.anthony.ies](http://www.anthony.ies)
* [http://www.tienda.ies](http://www.tienda.ies)
* [http://www.blog.ies](http://www.blog.ies)

---

🎓 **Si esto funciona: examen aprobado.**
