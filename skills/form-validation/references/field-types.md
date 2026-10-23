# Reglas por tipo de campo

Cada fila dice qué validar y cómo hacerlo en **servidor** (C#). La capa cliente replica la
misma regla con el control que uses (ver `telerik-aspx.md`, `aspnet-validators.md`,
`kendo.md` o `html5.md`).

> Todas las regex van ancladas con `^...$`. Sin anclas, `\d{5}` acepta `abc12345xyz`.
> Usa la sintaxis común a JavaScript y .NET (evita `(?i)`, lookbehind, grupos con nombre)
> porque los validators de ASP.NET ejecutan la misma expresión en ambos motores.

## Tabla resumen

| Tipo | Requerido | Tipo / parseo | Longitud | Rango | Formato |
|---|---|---|---|---|---|
| Texto libre | `!string.IsNullOrWhiteSpace(v)` | `string` (aplica `Trim()`) | `<= tamaño de la columna` (y mínimo si aplica) | — | Opcional: lista blanca de caracteres |
| Entero | sí/no según negocio | `int.TryParse(v, NumberStyles.Integer, CultureInfo.InvariantCulture, out n)` | — | `min <= n <= max` | Solo dígitos (y signo si se permite) |
| Decimal / moneda | sí/no | `decimal.TryParse(v, NumberStyles.Number, cultura, out d)` | — | `min <= d <= max` | Escala = `DECIMAL(p,s)` de la columna |
| Correo | sí/no | `string` | `<= 254` (o la columna) | — | Regex pragmática + confirmación por correo si importa |
| Teléfono (MX) | sí/no | `string` normalizado a dígitos | 10 | — | `^\d{10}$` |
| RFC | sí/no | `string` en mayúsculas | 12 (moral) o 13 (física) | — | ver abajo |
| CURP | sí/no | `string` en mayúsculas | 18 | — | ver abajo |
| Código postal (MX) | sí/no | `string` (no `int`: puede iniciar con 0) | 5 | — | `^\d{5}$` |
| Fecha | sí/no | `DateTime.TryParseExact(v, "dd/MM/yyyy", cultura, DateTimeStyles.None, out f)` | — | Mín/máx de negocio y del tipo SQL | Formato único documentado |
| Rango de fechas | ambos o ninguno | dos fechas parseadas | — | `inicio <= fin`, duración máxima si aplica | — |
| Lista / catálogo | valor distinto al "Seleccione…" | `int`/`string` | — | **El valor existe en el catálogo** | — |
| Archivo | sí/no | `HttpPostedFile` / archivo subido | Tamaño máximo en bytes | — | Extensión en lista blanca **y** contenido verificado |

## Detalle y "por qué"

### Texto libre
- Recorta (`Trim()`) antes de medir: "   " no es un nombre.
- Longitud máxima = tamaño de la columna (`NVARCHAR(n)`). Ver punto 3 de `SKILL.md`.
- **No** intentes "limpiar" HTML aquí: guarda el texto tal cual y **codifica al pintar**
  (`server-security.md`). Limpiar al guardar corrompe datos legítimos ("O'Brien", "<3").

### Numéricos
- Parsea con cultura explícita. En `es-MX` el separador decimal es `.`, en `es-ES` es `,`;
  si el servidor tiene otra cultura que el usuario, `1,5` puede convertirse en `15`.
- El rango lo define el negocio **y** el tipo SQL: `INT` llega a 2,147,483,647; `DECIMAL(10,2)`
  admite 8 dígitos enteros.
- Para "solo números" que **no** son cantidades (cuenta, folio, CP) usa `string` + regex,
  no `int`: los ceros a la izquierda importan.

### Correo
- Regex pragmática (no intentes cubrir el RFC 5322 completo):
  `^[^@\s]+@[^@\s]+\.[^@\s]+$`
- Longitud máxima práctica: 254 caracteres.
- Si el correo es crítico (recuperar contraseña), la única validación real es enviar un
  correo de confirmación.

### Teléfono, RFC, CURP, CP (México)
- Normaliza antes de validar: quita espacios/guiones del teléfono, pasa RFC/CURP a
  mayúsculas con `ToUpperInvariant()`.
- Teléfono (10 dígitos): `^\d{10}$`
- CP: `^\d{5}$`
- RFC (persona moral 3 letras, física 4; luego fecha AAMMDD y homoclave):
  `^[A-ZÑ&]{3,4}\d{6}[A-Z0-9]{3}$`
- CURP (estructura; no valida el dígito verificador):
  `^[A-Z][AEIOUX][A-Z]{2}\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])[HM](AS|BC|BS|CC|CL|CM|CS|CH|DF|DG|GT|GR|HG|JC|MC|MN|MS|NT|NL|OC|PL|QT|QR|SP|SL|SR|TC|TS|TL|VZ|YN|ZS|NE)[B-DF-HJ-NP-TV-Z]{3}[A-Z0-9]\d$`
- Estas regex validan **forma**, no existencia. Si el negocio exige que el RFC/CURP sea
  real, eso es una consulta a un servicio externo, no una regex.

### Fechas
- Un solo formato de entrada documentado (p. ej. `dd/MM/yyyy`) y `TryParseExact`.
- Límites de SQL Server: `DATETIME` solo admite desde 1753-01-01; `DATE` y `DATETIME2`
  desde 0001-01-01. Una fecha válida en .NET puede fallar al insertarse en `DATETIME`.
- Define mín/máx de negocio (p. ej. fecha de nacimiento: no futura, no antes de 1900).
- Si la hora no importa, compara solo `.Date` para evitar errores por la hora.

### Rango de fechas
- Ambas fechas presentes o ambas vacías (según negocio).
- `inicio <= fin` **en servidor**, aunque el cliente ya lo limite (p. ej. el picker de fin
  con `MinDate` = inicio).
- Duración máxima si aplica (`(fin - inicio).TotalDays <= 31`), para que nadie pida un
  reporte de 20 años que tumbe la base.

### Listas / catálogos
- El cliente solo ofrece opciones válidas, pero el POST puede traer cualquier valor.
- Valida en servidor que el `SelectedValue` exista en el catálogo (consulta o lista en
  memoria) y que el usuario tenga permiso para elegirlo.
- `EnableEventValidation` ayuda con controles estándar, pero no sustituye esta regla
  (ver `server-security.md`).

### Archivos
- Lista blanca de extensiones (`.pdf`, `.jpg`…), nunca lista negra.
- Tamaño máximo en servidor; recuerda que `maxRequestLength` (KB) y
  `maxAllowedContentLength` (bytes, IIS) también limitan la petición completa.
- La extensión y el `ContentType` los manda el cliente: verifica el contenido (firma /
  "magic number") si el archivo se va a procesar o servir.
- Guarda con un nombre generado por ti (GUID), fuera del directorio web o sin permisos de
  ejecución, y nunca uses el nombre original como ruta.
