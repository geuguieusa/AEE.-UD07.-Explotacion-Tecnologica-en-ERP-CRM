# MANUAL EXPLOTACION WillmanTech S.L. ERP/CRM
> Markdown realizado bajo la norma [ISO/IEC/IEEE 26514:2022](https://www.iso.org/standard/77451.html)

---

## 1. Introducción y Arquitectura

El sistema ERP de WillmanTech S.L. está basado en **Odoo 17 Community** y desplegado con **Docker Compose** sobre un servidor Linux. La aplicación corre en el contenedor `willmantech_web` (puerto 8069) y se conecta internamente a `willmantech_db`, que ejecuta **PostgreSQL 15**. PostgreSQL no está expuesto al exterior, solo Odoo accede a él.

El módulo personalizado `willmantech_invoices` incluye la plantilla `report_invoice_willmantech.xml`, que hereda la plantilla oficial de Odoo (`account.report_invoice_document`) y la modifica con bloques `xpath`.

**Módulos activos:** Facturación (`account`), Ventas (`sale_management`), CRM (`crm`).

**Topología:**
```
Host
├── willmantech_web → Odoo 17 (:8069)
└── willmantech_db  → PostgreSQL 15 (red interna)
```

**Stack:** Python 3.11 · wkhtmltopdf 0.12.6 · Docker Engine 24+

---

## 2. Instalación y Reinstalación

**Variables de entorno (`.env`):**
```dotenv
POSTGRES_DB=willmantech_db
POSTGRES_USER=odoo
POSTGRES_PASSWORD=CambiarEnProduccion!
ODOO_ADMIN_PASSWD=MasterKey_WillmanTech!
ODOO_PORT=8069
```

> No subir el `.env` al repositorio. Añadirlo al `.gitignore`.

**Arrancar el entorno:**
```bash
docker compose pull
docker compose up -d
# Acceder en: http://<IP>:8069
```

**Instalar el módulo personalizado:**
```bash
docker compose exec web odoo --update=willmantech_invoices --stop-after-init --database=willmantech_db
```

**Reinstalación completa** (borra todos los datos):
```bash
docker compose down -v && docker compose up -d
```

---

## 3. Seguridad y Control de Acceso

| Rol | Acceso |
|---|---|
| Administrador | Total — todos los módulos y configuración del sistema |
| Contable | Total en Facturación, solo lectura en Ventas |
| Comercial | Solo sus propios registros de Ventas y CRM |

Los roles se asignan desde **Ajustes → Usuarios y Empresas → Usuarios**, en la sección "Permisos de acceso" de cada usuario.

**Contraseñas:** mínimo 12 caracteres, mayúsculas + números + símbolo, caducidad 90 días, bloqueo tras 5 intentos fallidos.

**2FA obligatorio** para Administrador y Contable:
> *Ajustes → Seguridad → Autenticación de dos factores → Obligatoria*

---

## 4. Backup y Restauración

**Backup:**
```bash
mkdir -p backups
docker compose exec db pg_dump --username=odoo --format=custom \
    willmantech_db > backups/backup_$(date +%Y%m%d).dump
```

Se recomienda automatizar este comando con un cron diario y conservar al menos 7 copias.

**Restauración:**
```bash
docker compose stop web
docker compose exec db psql -U postgres -c "DROP DATABASE willmantech_db; CREATE DATABASE willmantech_db OWNER odoo;"
docker compose exec -T db pg_restore --username=odoo --dbname=willmantech_db < backups/backup_YYYYMMDD.dump
docker compose start web
```

---

## 5. Flujo de Facturación e Informes

**Crear una factura:** Facturación → Clientes → Facturas → Nuevo → rellenar cliente y líneas → **Confirmar** → **Imprimir**.

Al confirmar, Odoo asigna el número secuencial (`INV/2026/XXXX`) y bloquea la factura para edición. Al imprimir, el sistema genera el PDF siguiendo este pipeline:

**Pipeline HTML → wkhtmltopdf → PDF:**
```
Odoo carga los datos de la factura desde PostgreSQL
QWeb procesa report_invoice_willmantech.xml:
   - xpath aplica los cambios sobre la plantilla oficial
   - t-foreach itera sobre las líneas de la factura
   - t-if oculta la columna de descuento si no hay ninguno
   - t-field inyecta doc.name, doc.invoice_date y doc.amount_total
   → genera HTML con Bootstrap
wkhtmltopdf convierte el HTML a PDF (motor WebKit)
El PDF se sirve al navegador como descarga 
```

**Problemas frecuentes:**
- Si el PDF sale en blanco, verificar que wkhtmltopdf está instalado en el contenedor: `docker compose exec web wkhtmltopdf --version`
- Si el módulo no aparece en Odoo, repetir el comando `--update` de la sección 2.
- Para consultar errores en tiempo real: `docker compose logs -f web`