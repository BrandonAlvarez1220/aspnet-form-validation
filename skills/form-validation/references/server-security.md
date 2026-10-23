# Servidor: la validación real y la seguridad

El servidor es la única capa que controlas. Todo lo que llega en el POST (texto, valores
de combos, campos ocultos, ViewState, nombres de archivo) puede haber sido escrito a mano.

## 1. Page.IsValid antes de procesar

```csharp
protected void btnGuardar_Click(object sender, EventArgs e)
{
    Page.Validate("Alta");      // ejecuta los validators del grupo en servidor
    if (!Page.IsValid) return;  // nunca sigas si falló alguno
    // ...
}
```

- Los botones con `CausesValidation="true"` ya llaman a `Page.Validate(grupo)` antes del
  evento `Click`, pero llamarlo explícitamente documenta la intención y protege si alguien
  cambia el botón.
- En `Page_Load`, `Page.IsValid` no es fiable hasta llamar `Page.Validate(...)`.
- `Page.Validate()` sin argumento ignora los grupos.

## 2. Repite las reglas en code-behind

Los validators cubren requerido/tipo/rango/formato, pero hay reglas que solo existen en
servidor y deben estar ahí aunque el cliente "ya lo haga":

- Longitud contra la columna de SQL (`MaxLength` se salta).
- Que el valor del combo exista en el catálogo y el usuario tenga permiso sobre él.
- Rangos de los controles Telerik/Kendo (`MinValue`, `MaxDate`, …) que no tengan un
  validator ASP.NET asociado.
- Reglas cruzadas (inicio <= fin, total = suma de partidas).

Patrón recomendado: un método de validación que devuelva la lista de errores, reutilizable
desde la página, servicios o importaciones masivas.

```csharp
public static class Limites { public const int NombreMax = 100; }

private List<string> ValidarAlta(string nombre, DateTime? ini, DateTime? fin)
{
    var errores = new List<string>();
    nombre = (nombre ?? "").Trim();
    if (nombre.Length == 0)                 errores.Add("El nombre es obligatorio.");
    else if (nombre.Length > Limites.NombreMax) errores.Add($"El nombre admite máximo {Limites.NombreMax} caracteres.");
    if (!ini.HasValue || !fin.HasValue)     errores.Add("Las fechas inicio y fin son obligatorias.");
    else if (ini.Value.Date > fin.Value.Date) errores.Add("La fecha fin debe ser igual o posterior a la fecha inicio.");
    return errores;
}
```

Para mostrar errores de servidor junto a los demás, puedes agregar un `CustomValidator`
con `IsValid = false` y su `ErrorMessage` al `ValidationSummary`.

## 3. SqlParameter (nunca concatenar)

❌ Incorrecto (inyección SQL; `txtNombre.Text = "x'; DROP TABLE Clientes;--"`):

```csharp
var sql = "INSERT INTO Clientes (Nombre, FechaAlta) VALUES ('" + txtNombre.Text + "', '" + txtFecha.Text + "')";
using (var cmd = new SqlCommand(sql, cn)) { cmd.ExecuteNonQuery(); }
```

✅ Correcto (tipo y tamaño explícitos, iguales a la columna):

```csharp
const string sql = "INSERT INTO Clientes (Nombre, FechaAlta) VALUES (@Nombre, @FechaAlta)";
using (var cn = new SqlConnection(cadena))
using (var cmd = new SqlCommand(sql, cn))
{
    cmd.Parameters.Add("@Nombre", SqlDbType.NVarChar, Limites.NombreMax).Value = nombre;
    cmd.Parameters.Add("@FechaAlta", SqlDbType.Date).Value = fecha;          // DateTime ya validado
    cn.Open();
    cmd.ExecuteNonQuery();
}
```

- Prefiere `Parameters.Add(nombre, SqlDbType, tamaño)` a `AddWithValue`: `AddWithValue`
  infiere el tipo (p. ej. `nvarchar(4000)` contra una columna `varchar`), lo que puede
  provocar conversiones implícitas y planes de ejecución peores.
- Valores nulos: `(object)valor ?? DBNull.Value`.
- En procedimientos almacenados con SQL dinámico, el riesgo se mueve al SP: usa
  `sp_executesql` con parámetros, nunca `EXEC('...' + @param)`.
- Nombres de columna/orden dinámicos (ORDER BY) no se parametrizan: usa lista blanca.

## 4. Codificación de salida

La defensa contra XSS es **codificar al pintar**, según el contexto:

| Contexto | Cómo |
|---|---|
| Texto en HTML (ASPX) | `<%: valor %>` (ASP.NET 4+) |
| Data binding | `<%#: Item.Nombre %>` o `<%#: Eval("Nombre") %>` (ASP.NET 4.5+) |
| Code-behind a un `Literal`/`Label.Text` | `HttpUtility.HtmlEncode(valor)` |
| Atributo HTML (siempre entre comillas dobles) | `HttpUtility.HtmlAttributeEncode(valor)` |
| Dentro de JavaScript | `HttpUtility.JavaScriptStringEncode(valor)` |
| En una URL | `HttpUtility.UrlEncode(valor)` |

```aspx
<%-- ❌ --%> <p>Hola <%= Request.QueryString["nombre"] %></p>
<%-- ✅ --%> <p>Hola <%: Request.QueryString["nombre"] %></p>
```

- `<%= %>` y `<%# %>` **no** codifican.
- `Label.Text` **no** codifica. `Literal` codifica solo con `Mode="Encode"` (los otros
  valores de `LiteralMode` son `PassThrough` y `Transform`); fíjalo explícito en vez de
  confiar en el valor por defecto. `TextBox.Text` sí se codifica al renderizar.
- Columnas de grids: revisa si el control codifica (`HtmlEncode` en `BoundField`, que en
  `asp:GridView` es `true` por defecto). En RadGrid, comprueba la propiedad equivalente de
  la columna en tu versión **⚠ sin verificar**.
- `AntiXssEncoder` (`System.Web.Security.AntiXss`) usa lista blanca y puede registrarse
  como codificador por defecto con `<httpRuntime encoderType="..." />`.
- Filtrar `<script>` al guardar es, como mucho, una capa extra: hay muchas formas de
  ejecutar script sin esa etiqueta (`onerror=`, `javascript:`).

## 5. Request validation

ASP.NET rechaza por defecto entradas que parecen marcado (`<x`, `&#`) con
`HttpRequestValidationException` ("A potentially dangerous Request.Form value…").

- `<httpRuntime requestValidationMode="4.5" />` hace la validación diferida (solo al leer
  el valor). Cualquier valor numérico menor a 4.0 se interpreta como 2.0.
- Desactivar por control (4.5+): `ValidateRequestMode="Disabled"` en el control que
  necesita HTML (p. ej. el que hospeda un editor), no en toda la página ni en `web.config`.
- Leer un valor sin validar: `Request.Unvalidated.Form["campo"]`.
- **Es una red de seguridad, no la defensa.** No detecta todo y no protege datos que
  entran por otras vías (importaciones, APIs). La defensa sigue siendo codificar al pintar.
- Si la desactivas para un campo, ese campo **debe** sanitizarse en servidor y codificarse
  al pintar.

## 6. EventValidation y ViewState

- `EnableEventValidation` (por defecto `true`) rechaza postbacks con eventos o valores que
  el control no pudo haber generado (p. ej. un valor de `DropDownList` que no estaba en la
  lista). No lo desactives para "arreglar" un error de callback; registra el valor con
  `ClientScript.RegisterForEventValidation` o corrige el origen.
  Ayuda, pero **no** sustituye la validación de catálogo del punto 2 (controles de terceros
  o con texto libre pueden no participar).
- `EnableViewStateMac`: desde ASP.NET 4.5.2 el runtime lo trata siempre como `true` aunque
  se configure `false`. Nunca lo pongas en `false`.
- `ViewState` no es un almacén seguro: está firmado, no necesariamente cifrado. No guardes
  precios, permisos ni datos sensibles; vuelve a leerlos de la base en el postback.
- CSRF: asigna `ViewStateUserKey` en `Page_Init`/`OnInit` (p. ej. en una página base o
  `Site.Master`) a un valor único por sesión/usuario (regla CA5368). Las plantillas de
  Web Forms con autenticación ya incluyen un token anti-XSRF en `Site.Master.cs`.

```csharp
protected override void OnInit(EventArgs e)
{
    base.OnInit(e);
    if (User.Identity.IsAuthenticated) ViewStateUserKey = Session.SessionID;
}
```

## 7. Excepciones justificadas

Hay casos en que el campo **debe** aceptar HTML (RadEditor, descripciones enriquecidas):

1. Desactiva request validation **solo** en ese control (`ValidateRequestMode="Disabled"`).
2. Mantén los filtros de contenido del editor activos (ver `telerik-aspx.md` → RadEditor).
3. Sanitiza en servidor con lista blanca (librería de sanitización), no con regex.
4. Limita la longitud del HTML en servidor.
5. Pinta el resultado sanitizado sin codificar **solo** en ese punto y documenta por qué.

Cualquier otra excepción (desactivar `EnableEventValidation`, request validation global)
requiere un comentario en el código con el motivo y la compensación en servidor.

## 8. Errores al usuario

- Mensajes genéricos ante excepciones ("No fue posible guardar. Intenta de nuevo."),
  detalle solo en el log.
- `<customErrors mode="RemoteOnly" />` (o `On`) en producción; nunca la pantalla amarilla
  con la traza.

## Fuentes

- Page.Validate: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.page.validate?view=netframework-4.8.1
- Page.IsValid: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.page.isvalid?view=netframework-4.8.1
- Request Validation in ASP.NET: https://learn.microsoft.com/en-us/previous-versions/aspnet/hh882339(v=vs.110)
- HttpRequest.Unvalidated: https://learn.microsoft.com/en-us/dotnet/api/system.web.httprequest.unvalidated?view=netframework-4.8.1
- Control.ValidateRequestMode: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.control.validaterequestmode?view=netframework-4.8.1
- httpRuntime element: https://learn.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/e1f13641(v=vs.100)
- @ Page (EnableEventValidation): https://learn.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/ydy4x04a(v=vs.100)
- Farewell, EnableViewStateMac!: https://devblogs.microsoft.com/dotnet/farewell-enableviewstatemac/
- What not to do in ASP.NET: https://learn.microsoft.com/en-us/aspnet/aspnet/overview/web-development-best-practices/what-not-to-do-in-aspnet-and-what-to-do-instead
- CA5368 ViewStateUserKey: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca5368
- What's New in Web Forms in ASP.NET 4.5 (`<%#: %>`): https://learn.microsoft.com/en-us/aspnet/web-forms/overview/getting-started/hands-on-labs/whats-new-in-web-forms-in-aspnet-45
- ASP.NET Security (MSDN Magazine, `<%: %>`): https://learn.microsoft.com/en-us/archive/msdn-magazine/2012/january/asp-net-security-securing-your-asp-net-applications
- HttpUtility.HtmlEncode: https://learn.microsoft.com/en-us/dotnet/api/system.web.httputility.htmlencode
- HttpUtility.HtmlAttributeEncode: https://learn.microsoft.com/en-us/dotnet/api/system.web.httputility.htmlattributeencode
- HttpUtility.JavaScriptStringEncode: https://learn.microsoft.com/en-us/dotnet/api/system.web.httputility.javascriptstringencode
- LiteralMode: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.literalmode?view=netframework-4.8.1
- BoundField.HtmlEncode: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.boundfield.htmlencode?view=netframework-4.8.1
- ClientScriptManager.RegisterForEventValidation: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.clientscriptmanager.registerforeventvalidation?view=netframework-4.8.1
- AntiXssEncoder: https://learn.microsoft.com/en-us/dotnet/api/system.web.security.antixss.antixssencoder?view=netframework-4.8.1
