# Kendo UI for jQuery en páginas ASPX

> Mismo enfoque que el resto: los widgets Kendo son **solo cliente**. Lo que limitan
> (`min`, `max`, `decimals`) y lo que valida el Kendo Validator se repite en el
> code-behind (`server-security.md`).
>
> Los nombres se contrastaron con resultados de búsqueda restringidos a docs.telerik.com y
> telerik.com (no se pudieron abrir las páginas directamente). Lo no confirmado está
> marcado **⚠ sin verificar**.

## Cómo llega el valor al servidor

Kendo decora un `<input>` existente; en el postback llega el **texto del input** con su
`name`, no un objeto. En Web Forms:

```aspx
<asp:TextBox ID="txtInicio" runat="server" ClientIDMode="Static" />
<asp:TextBox ID="txtFin"    runat="server" ClientIDMode="Static" />
<asp:TextBox ID="txtMonto"  runat="server" ClientIDMode="Static" />
```

`ClientIDMode="Static"` hace que el `id` en HTML sea `txtInicio`, así el selector de jQuery
es estable. En servidor lees `txtInicio.Text` y lo parseas con el **mismo formato** que
configuraste en el widget.

## DatePicker

Opciones: `min`, `max`, `format` **⚠ sin verificar** (formato de despliegue, p. ej.
`"dd/MM/yyyy"`), `change` (evento). Métodos: `value()` (obtiene/asigna la fecha),
`min()` / `max()` (obtienen/asignan el límite), `setOptions()`.

Si el usuario teclea una fecha fuera de rango o inválida, `value()` devuelve `null`; el
texto tecleado sigue en el input y **se envía tal cual** en el POST.

### Rango: el inicio limita el mínimo del fin

```js
$(function () {
  var fin = $("#txtFin").kendoDatePicker({
    min: new Date(2020, 0, 1), max: new Date(2030, 11, 31), format: "dd/MM/yyyy"
  }).data("kendoDatePicker");

  $("#txtInicio").kendoDatePicker({
    min: new Date(2020, 0, 1), max: new Date(2030, 11, 31), format: "dd/MM/yyyy",
    change: function () {
      var inicio = this.value();
      if (inicio) {
        fin.min(inicio);
        if (fin.value() && fin.value() < inicio) { fin.value(null); }
      }
    }
  });
});
```

Alternativa documentada: `fin.setOptions({ min: inicio })`. Ojo: `setOptions` puede
restablecer el valor al inicial si no pasas `value`; para solo mover el límite, `min()`
es más directo.

Servidor:

```csharp
var es = CultureInfo.GetCultureInfo("es-MX");
bool okIni = DateTime.TryParseExact(txtInicio.Text, "dd/MM/yyyy", es, DateTimeStyles.None, out var ini);
bool okFin = DateTime.TryParseExact(txtFin.Text,    "dd/MM/yyyy", es, DateTimeStyles.None, out var fin);
if (!okIni || !okFin)        { /* "Captura fechas válidas (dd/mm/aaaa)." */ }
else if (ini > fin)          { /* "La fecha fin debe ser igual o posterior a la fecha inicio." */ }
else if (ini < new DateTime(2020, 1, 1) || fin > new DateTime(2030, 12, 31)) { /* fuera de rango */ }
```

## NumericTextBox

Opciones: `min`, `max`, `decimals`, `restrictDecimals` (por defecto `false`; con `true`
impide teclear más decimales de los indicados por `decimals`), `format` (p. ej. `"n0"`,
`"n2"`, `"c2"`), `round` (por defecto redondea "half-up"; con `false` trunca). El valor se
ajusta a `min`/`max` al perder el foco.

Para "solo enteros": `decimals: 0`, `restrictDecimals: true`, `format: "n0"`.

```js
$("#txtMonto").kendoNumericTextBox({
  min: 0.01, max: 99999.99, decimals: 2, restrictDecimals: true, format: "c2"
});
```

Cuidado: el input visible muestra el valor **formateado** (`$1,234.50`). Qué texto se
envía en el POST (formateado o crudo) **⚠ sin verificar**; Kendo crea un input visible
adicional y mantiene el original con el valor numérico. Valida en servidor aceptando
solo el formato numérico esperado:

```csharp
if (!decimal.TryParse(txtMonto.Text, NumberStyles.Number, CultureInfo.InvariantCulture, out var monto)
    || monto < 0.01m || monto > 99999.99m || decimal.Round(monto, 2) != monto)
{ /* "El monto debe estar entre $0.01 y $99,999.99 con máximo 2 decimales." */ }
```

Números con más de ~16 dígitos pierden precisión en JavaScript (`Number`): para claves o
folios largos usa un input de texto + regex, no NumericTextBox.

## Validator

`kendoValidator` respeta los atributos HTML5 (`required`, `pattern`, `type="email"`,
`min`/`max`) y agrega reglas propias. Las restricciones HTML5 se evalúan **antes** que las
reglas personalizadas.

- Configuración: `rules`, `messages`, `errorTemplate` (un solo elemento raíz; no se puede
  definir por input), `validationSummary`.
- Métodos: `validate()`, `validateInput(input)`, `errors()`, `reset()`, `setOptions()`,
  `showValidationSummary()`, `hideValidationSummary()`.
- Prioridad de mensajes por campo: `data-[regla]-msg` → `validationMessage` → `title`.
  Ej.: `data-required-msg="El correo es obligatorio."`.

```aspx
<div id="frmAlta">
  <asp:TextBox ID="txtCorreo" runat="server" ClientIDMode="Static" TextMode="Email"
      MaxLength="254" required="required" data-required-msg="El correo es obligatorio."
      data-email-msg="El correo no tiene un formato válido." />
  <asp:TextBox ID="txtCp" runat="server" ClientIDMode="Static" MaxLength="5"
      required="required" pattern="\d{5}"
      data-required-msg="El código postal es obligatorio."
      data-pattern-msg="El código postal debe tener 5 dígitos." />
  <asp:Button ID="btnGuardar" runat="server" Text="Guardar" ClientIDMode="Static"
      OnClick="btnGuardar_Click" />
</div>

<script>
$(function () {
  var validator = $("#frmAlta").kendoValidator({
    rules: {
      rangoFechas: function (input) {
        if (!input.is("#txtFin")) return true;
        var ini = $("#txtInicio").data("kendoDatePicker"),
            fin = $("#txtFin").data("kendoDatePicker");
        return !ini.value() || !fin.value() || fin.value() >= ini.value();
      }
    },
    messages: {
      rangoFechas: "La fecha fin debe ser igual o posterior a la fecha inicio."
    }
  }).data("kendoValidator");

  $("#btnGuardar").on("click", function (e) {
    if (!validator.validate()) { e.preventDefault(); }   // evita el postback
  });
});
</script>
```

- `data-email-msg` y `data-pattern-msg` siguen la convención `data-[regla]-msg`; los
  nombres exactos de regla `email` y `pattern` **⚠ sin verificar**.
- `e.preventDefault()` solo evita el postback desde ese botón. Enter en otro campo, otro
  botón o una petición fabricada saltan el Validator: el servidor valida igual.
- Si la página también tiene validators de ASP.NET, coordina ambos (o elige uno) para no
  mostrar dos mensajes por campo.

Servidor: mismas reglas en `btnGuardar_Click` con `Page.IsValid` + validación propia
(`server-security.md`).

## Cultura

Para fechas y números en español, carga el archivo de cultura de Kendo y llama a
`kendo.culture("es-MX")` antes de crear los widgets **⚠ sin verificar** (nombre y ruta
del script de cultura según tu versión). Usa la misma cultura en `TryParse` del servidor.

## Fuentes

- DatePicker API: https://docs.telerik.com/kendo-ui/api/javascript/ui/datepicker
- DatePicker min(): https://docs.telerik.com/kendo-ui/api/javascript/ui/datepicker/methods/min
- DatePicker setOptions(): https://www.telerik.com/kendo-jquery-ui/documentation/api/javascript/ui/datepicker/methods/setoptions
- KB: restringir entrada a min/max: https://www.telerik.com/kendo-jquery-ui/documentation/knowledge-base/restrict-user-input-to-minmax-values
- KB: min/max global: https://docs.telerik.com/kendo-ui/knowledge-base/datepicker-global-min-max-range
- KB: min dinámico en DateTimePicker (patrón de rango): https://www.telerik.com/kendo-jquery-ui/documentation/knowledge-base/dynamically-set-min-date-time-kendo-datetimepicker
- NumericTextBox API: https://docs.telerik.com/kendo-ui/api/javascript/ui/numerictextbox
- NumericTextBox restrictDecimals: https://docs.telerik.com/kendo-ui/api/javascript/ui/numerictextbox/configuration/restrictdecimals
- NumericTextBox input restrictions: https://www.telerik.com/kendo-jquery-ui/documentation/controls/numerictextbox/precision
- KB: restringir decimales: https://www.telerik.com/kendo-jquery-ui/documentation/knowledge-base/how-to-restrict-decimals-kendo-numeric-text-box
- Validator API: https://www.telerik.com/kendo-jquery-ui/documentation/api/javascript/ui/validator
- Validator rules: https://www.telerik.com/kendo-jquery-ui/documentation/controls/validator/rules
- Validator errorTemplate: https://www.telerik.com/kendo-jquery-ui/documentation/api/javascript/ui/validator/configuration/errortemplate
- Validator validateInput: https://docs.telerik.com/kendo-ui/api/javascript/ui/validator/methods/validateinput
- Blog: reglas personalizadas del Validator: https://www.telerik.com/blogs/extending-the-kendo-ui-validator-with-custom-rules
