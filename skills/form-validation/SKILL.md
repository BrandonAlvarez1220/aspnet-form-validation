---
name: form-validation
description: Use when creating or reviewing forms in ASP.NET Web Forms (ASPX) with Telerik UI for ASP.NET AJAX (RadTextBox, RadNumericTextBox, RadDatePicker, RadButton, RadGrid, etc.), Kendo UI or HTML5. Applies a field-validation checklist (date ranges, max length, numeric-only, email format, required) in three layers: client, server and output encoding.
---

# Validación de formularios ASP.NET Web Forms

Usa esta skill al **crear o revisar** formularios ASPX (con Telerik UI for ASP.NET AJAX,
Kendo UI o HTML5 plano) y su code-behind en C# contra SQL Server.

## 1. Regla de las tres capas

| Capa | Para qué sirve | Dónde vive |
|---|---|---|
| **Cliente** | UX: avisar rápido, evitar postbacks inútiles | Propiedades del control, validators de ASP.NET, Kendo Validator, atributos HTML5 |
| **Servidor** | **La validación real.** Es la única que el usuario no puede saltarse | `Page.IsValid` + reglas repetidas en C# + `SqlParameter` |
| **Salida** | Que lo guardado no se ejecute al pintarse (XSS) | `<%: %>`, `<%#: %>`, `HttpUtility.HtmlEncode` |

**Ninguna regla existe solo en el cliente.** Cualquier petición se puede fabricar sin
navegador (curl, Postman, DevTools), así que todo lo que valida el cliente se repite en C#.
Las reglas de cliente son una copia de cortesía de las del servidor, no al revés.

## 2. Checklist por campo

Para **cada** campo del formulario, decide y deja explícito:

1. **Requerido**: ¿puede ir vacío? Recorta espacios (`Trim()`) antes de decidir.
2. **Tipo**: texto, entero, decimal, fecha, correo, catálogo, archivo. Parsea con
   `TryParse`/`TryParseExact` y una `CultureInfo` explícita, nunca con `Convert.*` a ciegas.
3. **Longitud**: mínimo y máximo de caracteres.
4. **Rango**: mínimo/máximo para números y fechas; para rangos de fechas, `inicio <= fin`.
5. **Formato**: patrón (correo, teléfono, RFC, CURP, CP) con regex anclada `^...$`.

Si el campo es de catálogo (combo, radio), valida que el valor recibido **exista en el
catálogo**, no solo que no venga vacío.

## 3. MaxLength alineado con SQL Server

El `MaxLength` del control debe ser **igual** al tamaño de la columna destino:

- `NVARCHAR(100)` → `MaxLength="100"` y en servidor `valor.Length <= 100`.
- `VARCHAR(n)` cuenta **bytes** según la intercalación (collation); `NVARCHAR(n)` cuenta
  pares de bytes (unidades UTF-16), igual que `string.Length` en .NET.
- Si cambias la columna, cambia el control **y** la constante del servidor. Centraliza el
  número en una constante (p. ej. `Limites.NombreMax = 100`) para no tener tres copias.

¿Por qué? Si el control permite más que la columna, SQL Server lanza
"String or binary data would be truncated" o, peor, se trunca en silencio en código heredado.

## 4. Mensajes de error

- En **español**, junto al campo (validator con `Display="Dynamic"` al lado del control) y,
  si el formulario es largo, también en un `ValidationSummary`.
- Formato consistente: qué está mal + cómo corregirlo. Ej.: "La fecha fin debe ser igual o
  posterior a la fecha inicio."
- El mismo texto en cliente y servidor; no muestres excepciones ni detalles de SQL al usuario.

## 5. Lo que NO se debe hacer

- ❌ Confiar en `MaxLength`, `MinValue`, `MaxDate`, `required` o JavaScript como seguridad.
  Son UX; se saltan editando el HTML o enviando la petición a mano.
- ❌ Concatenar SQL con texto del usuario (`"... WHERE Id = " + txtId.Text`). Usa siempre
  `SqlParameter` con tipo y tamaño.
- ❌ Pintar datos con `<%= %>` o `Eval()` sin codificar. Usa `<%: %>` / `<%#: %>`.
- ❌ Filtrar `<script>` (o listas negras de etiquetas) como defensa principal contra XSS.
  Como mucho es una capa extra; la defensa es **codificar al pintar**.
- ❌ Procesar el postback sin comprobar `Page.IsValid`.
- ❌ Desactivar request validation o `EnableEventValidation` "para que funcione" sin
  documentar por qué y sin compensar en servidor.

## 6. Qué referencia leer

Lee **solo** el archivo que aplique al caso (están en `references/`):

| Caso | Archivo |
|---|---|
| Decidir la regla de un tipo de campo (texto, número, correo, RFC/CURP/CP, fechas, archivos…) | `references/field-types.md` |
| El formulario usa controles Telerik (`telerik:Rad*`) | `references/telerik-aspx.md` |
| Validators estándar (`asp:RequiredFieldValidator`, `CompareValidator`, `CustomValidator`, grupos) | `references/aspnet-validators.md` |
| Code-behind, acceso a datos, `Page.IsValid`, XSS, request validation, ViewState | `references/server-security.md` |
| Inputs HTML5 planos (`required`, `pattern`, `type="email"`) | `references/html5.md` |
| Widgets Kendo UI (DatePicker, NumericTextBox, Validator) | `references/kendo.md` |

Casi siempre necesitarás `server-security.md` además del archivo de la capa cliente.

## 7. Cómo entregar el trabajo

Al **crear** un formulario: entrega el ASPX con la validación de cliente **y** el
code-behind con la validación equivalente, `Page.IsValid` y `SqlParameter`.

Al **revisar** un formulario: recorre el checklist del punto 2 campo por campo y reporta
en una tabla `Campo | Regla esperada | Cliente | Servidor | Hallazgo`. Señala primero lo
que falte en servidor o en la salida (riesgo de seguridad) y después lo de UX.

Si una referencia marca un nombre de API como "⚠ sin verificar", avísalo al usuario
antes de usarlo y sugiere comprobarlo contra su versión del control.
