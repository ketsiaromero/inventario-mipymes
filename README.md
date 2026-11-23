# Sistema web de inventario y ventas para mipymes comerciales de Palmira

Prototipo funcional (TRL 5) del proyecto de grado de Ingeniería de Sistemas, UNAD.
Aplicación web responsiva, pensada para usarse desde el celular, que controla
productos, lotes con fecha de vencimiento (criterio FEFO), ventas, alertas de
reabastecimiento y reportes.

**Tecnología:** Python 3.12+, Django 5.2 LTS, PostgreSQL 14+, HTML5/CSS3 responsivo y JavaScript liviano.

## Funcionalidades

| Requerimiento | Dónde está |
|---|---|
| RF01 Productos y categorías | Inventario → Nuevo producto / Editar; Catálogos |
| RF02 Entradas por lote con vencimiento | Inventario → Entrada de lote |
| RF03 Venta desde el celular | Vender (búsqueda por nombre o código) |
| RF04 Descuento FEFO | `inventario/services.py` → `registrar_venta` |
| RF05 Impedir ventas sin existencias | Validación en servicio + restricción `CHECK` en PostgreSQL |
| RF06 Ajustes con motivo | Detalle del producto → Ajustar (administrador) |
| RF07 Alertas de mínimo | Inicio y Alertas |
| RF08 Alertas de vencimiento (N días) | Alertas (3, 7, 15 o 30 días; por defecto `ALERTA_DIAS_VENCIMIENTO`) |
| RF09 Anular venta con motivo | Detalle de la venta (administrador) |
| RF10 Reportes de ventas y margen | Reportes |
| RF11 Valor del inventario a costo | Reportes |
| RF12 Roles administrador y vendedor | Reportes → Usuarios |
| RF13 Código de barras con cámara | Vender → Escanear (navegadores con `BarcodeDetector`, p. ej. Chrome en Android) |
| RF14 Exportar CSV | Reportes → Descargar CSV |

### Reglas de negocio

- **FEFO:** sale primero el lote que vence antes; los lotes sin vencimiento salen al final; si dos lotes vencen el mismo día, sale primero el que ingresó antes.
- **Lotes vencidos:** no se venden. Aparecen en Alertas y se retiran con un ajuste por vencimiento.
- **Concurrencia:** cada venta corre en una transacción atómica que bloquea (`SELECT ... FOR UPDATE`) los productos en orden de id y luego sus lotes. Dos ventas simultáneas del último producto no pueden dejar existencias negativas; hay una prueba automatizada que lo verifica.
- **Anulación:** devuelve cada unidad al lote exacto del que salió (tabla `DetalleVentaLote`).
- **Permisos:** el vendedor vende, consulta inventario y alertas, y registra entradas. Solo el administrador crea o edita productos, hace ajustes, anula ventas y ve reportes y usuarios.

## Arquitectura

Monolito Django con patrón MVT y una capa de servicios separada de las vistas:

```
config/            configuración (variables de entorno, seguridad)
cuentas/           usuario con rol
inventario/
  models.py        modelo de datos (Figura 2 del documento)
  services.py      reglas que modifican existencias (transaccionales)
  selectors.py     consultas de solo lectura: alertas y reportes
  views.py         vistas (control de acceso por rol)
  templates/       pantallas mobile-first
  tests/           pruebas automatizadas
static/            CSS y JavaScript del punto de venta
```

## Instalación local

Requisitos: Python 3.12 o superior y PostgreSQL.

```bash
git clone <url-del-repositorio>
cd inventario-mipymes
python -m venv .venv
source .venv/bin/activate        # En Windows: .venv\Scripts\activate
pip install -r requirements.txt

# Crear la base de datos (una sola vez)
createdb inventario              # o desde pgAdmin

# Variables de entorno (ajuste usuario y clave de PostgreSQL)
export DJANGO_DEBUG=1
export DATABASE_URL=postgres://postgres:SU_CLAVE@localhost:5432/inventario
# En Windows PowerShell:  $env:DJANGO_DEBUG="1"; $env:DATABASE_URL="postgres://..."

python manage.py migrate
python manage.py cargar_demo --clave-admin "ClaveAdmin-2026" --clave-vendedor "ClaveVende-2026"
python manage.py runserver 0.0.0.0:8000
```

Abra `http://localhost:8000` e ingrese con `admin` o `vendedor`. Para probar
desde el celular en la misma red wifi, agregue la IP del computador a
`DJANGO_ALLOWED_HOSTS` y abra `http://IP-DEL-PC:8000`.

`cargar_demo` borra productos, lotes y ventas y crea datos de ejemplo. **No lo
ejecute en el comercio piloto:** allí se crea el administrador con
`python manage.py createsuperuser` y se cargan los productos reales.

## Pruebas

```bash
python manage.py test inventario
```

27 pruebas: orden FEFO, ventas que toman de varios lotes, lotes vencidos,
reversión completa ante falta de existencias, anulación, ajustes,
restricciones `CHECK`, concurrencia entre dos ventas simultáneas, permisos por
rol y reportes. Requieren PostgreSQL.

## Despliegue en Render

1. Suba el repositorio a GitHub.
2. En Render: **New → Blueprint** y seleccione el repositorio. `render.yaml`
   crea la base de datos PostgreSQL y el servicio web, y genera la clave secreta.
3. Cuando termine, abra la consola del servicio (**Shell**) y ejecute
   `python manage.py createsuperuser`. Después, en la aplicación, cambie el rol
   de ese usuario a Administrador en Reportes → Usuarios si hace falta.

El servidor exige HTTPS, usa cookies seguras y guarda las contraseñas con
hash (PBKDF2). Para las copias de seguridad diarias (RNF05) use las copias
automáticas del proveedor o programe `pg_dump`.

## Datos personales

El sistema no registra datos de clientes. Guarda nombre y usuario de los
operadores y nombre y teléfono de los proveedores. Ver la pantalla
*Tratamiento de datos personales* (Ley 1581 de 2012), que el comercio piloto
debe completar con sus datos.
