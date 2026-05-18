# MANUAL EXPLOTACION WillmanTechS.L ERP/CRM
> Markdown realizado bajo la norma [ISO/IEC/IEEE 26514:2022](https://www.iso.org/standard/77451.html)
---
## 1. Introducción y Arquitectura

El sistema ERP de WillmanTech S.L. está basado en **Odoo** y desplegado con **Docker Compose**.

El módulo personalizado de facturación incluye la plantilla `report_invoice_willmantech.xml`, que hereda la plantilla oficial de Odoo (`account.report_invoice_document`) y la adapta mediante bloques `xpath`.

**Módulos activos:** Facturación (`account`), Ventas (`sale_management`), CRM (`crm`).

**Topología:**
```
Host
├── willmantech_web → Odoo (puerto 8069)
└── willmantech_db  → PostgreSQL (red interna)
```

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

**Arrancar el entorno:**
```bash
cp .env.example .env        # editar con valores reales
docker compose pull
docker compose up -d
# Acceder en: http://<IP>:8069
```

**Instalar el módulo personalizado:**
```bash
docker compose exec web odoo --update=willmantech_invoices --stop-after-init
```

**Reinstalación completa** (borra datos):
```bash
docker compose down -v && docker compose up -d
```

---

## 3. Seguridad y Control de Acceso

| Rol | Acceso |
|---|---|
| Administrador | Total — todos los módulos |
| Contable | Total en Facturación, limitado en resto |
| Comercial | Solo Ventas y CRM propios |

**Contraseñas:** mínimo 12 caracteres, mayúsculas + números + símbolo, caducidad 90 días, bloqueo tras 5 intentos.

**2FA obligatorio** para Administrador y Contable:
> *Ajustes → Seguridad → Autenticación de dos factores → Obligatoria*

---

## 4. Backup y Restauración

**Backup:**
```bash
docker compose exec db pg_dump --username=odoo --format=custom \
    willmantech_db > backups/backup_$(date +%Y%m%d).dump
```

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

**Cómo funciona la plantilla `report_invoice_willmantech.xml`:**

El archivo hereda la plantilla oficial de Odoo con `inherit_id="account.report_invoice_document"` y la modifica mediante selectores `xpath`. Cada `xpath` apunta a un elemento concreto de la plantilla original y lo sustituye con `position="replace"`.

Los cambios aplicados son:

- **Título:** el bloque `//span[@name='invoice_title']` se reemplaza para mostrar "Factura WillmanTech S.L." seguido del número de factura mediante `t-field="o.name"`.
- **Fecha de emisión:** el bloque `//div[@name='invoice_date']` se reemplaza para mostrar la etiqueta en español y el campo `o.invoice_date` con `t-field`.
- **Cabeceras de la tabla:** los `th` de descripción, cantidad, precio, descuento y subtotal se traducen al español. La columna de descuento mantiene el `t-if="display_discount"` de la plantilla original, que la oculta automáticamente si ninguna línea tiene descuento.
- **Total neto:** se añade una fila adicional tras el importe pendiente que muestra `o.amount_total` con el widget monetario.


