# Telerik UI for ASP.NET AJAX: validación por control

> **⚠ Nota de versión.** El autor no especificó su versión de Telerik UI for ASP.NET AJAX.
> Este archivo se redactó contra la documentación más reciente disponible (la última
> versión localizada fue **2026.3.812, 2026 Q3**) y **no se contrastó con la versión del
> autor**. Si tu proyecto usa una versión anterior, comprueba las propiedades marcadas y
> revisa las notas de versión.
>
> Además, al redactarlo no se pudieron abrir directamente las páginas de docs.telerik.com;
> los nombres se contrastaron con los resultados de búsqueda restringidos a
> docs.telerik.com y telerik.com (ver "Fuentes"). Lo que no se pudo confirmar está marcado
> **⚠ sin verificar**.

Regla general: los controles Telerik restringen la **entrada** (UX). Todo lo que limitan
se vuelve a comprobar en el code-behind. Los validators estándar de ASP.NET
(`RequiredFieldValidator`, `CompareValidator`, `CustomValidator`…) funcionan con los
controles `Rad*` en cliente y servidor.

En los ejemplos se asume `<%@ Register TagPrefix="telerik" Namespace="Telerik.Web.UI" Assembly="Telerik.Web.UI" %>`
y un `ValidationGroup="Alta"` común.

---

## RadTextBox

Propiedades útiles: `MaxLength` (solo modo SingleLine/Password; ver nota), `TextMode`,
`EmptyMessage` (marcador visual, **no** es un valor ni una validación), `InputType`
(solo tiene efecto con `TextMode="SingleLine"`).

RadTextBox no tiene una propiedad "requerido": se usa `RequiredFieldValidator`.

```aspx
<telerik:RadTextBox ID="txtNombre" runat="server" MaxLength="100"
    EmptyMessage="Nombre completo" />
<asp:RequiredFieldValidator ID="rfvNombre" runat="server" ControlToValidate="txtNombre"
    ErrorMessage="El nombre es obligatorio." Display="Dynamic" ValidationGroup="Alta" />
```

Servidor:

```csharp
string nombre = (txtNombre.Text ?? "").Trim();
if (nombre.Length == 0)  { /* "El nombre es obligatorio." */ }
if (nombre.Length > 100) { /* "El nombre admite máximo 100 caracteres." */ }
```

- Si usas `RequiredFieldValidator.InitialValue`, debe ser un valor que el control pueda
  contener; un `InitialValue` más largo que `MaxLength` nunca coincide y el validator no falla.
- Con `TextMode="MultiLine"` el `MaxLength` no se aplica igual que en una sola línea
  (en HTML, `maxlength` en `<textarea>` depende del navegador) **⚠ sin verificar** cómo lo
  resuelve RadTextBox en tu versión: valida la longitud siempre en servidor.

## RadNumericTextBox

Propiedades: `MinValue`, `MaxValue`, `NumberFormat-DecimalDigits`,
`NumberFormat-AllowRounding`, `AllowOutOfRangeAutoCorrect` (por defecto `true`: corrige
el valor fuera de rango; con `false` lo deja y aplica el estilo inválido), `Type`
(valores `Number`, `Currency`, `Percent` **⚠ sin verificar**).

En servidor, `Value` es `double?` (nullable). Para montos usa `decimal` y convierte con
cuidado: `double` pierde precisión con números grandes.

```aspx
<telerik:RadNumericTextBox ID="ntbCantidad" runat="server"
    MinValue="1" MaxValue="999" NumberFormat-DecimalDigits="0"
    AllowOutOfRangeAutoCorrect="false" />
<asp:RequiredFieldValidator ID="rfvCantidad" runat="server" ControlToValidate="ntbCantidad"
    ErrorMessage="La cantidad es obligatoria." Display="Dynamic" ValidationGroup="Alta" />
<asp:RangeValidator ID="rvCantidad" runat="server" ControlToValidate="ntbCantidad"
    Type="Integer" MinimumValue="1" MaximumValue="999"
    ErrorMessage="La cantidad debe estar entre 1 y 999." Display="Dynamic" ValidationGroup="Alta" />
```

Servidor:

```csharp
if (!ntbCantidad.Value.HasValue)                     { /* obligatorio */ }
double v = ntbCantidad.Value.Value;
if (v < 1 || v > 999 || v != Math.Floor(v))          { /* fuera de rango o con decimales */ }
int cantidad = (int)v;
```

¿Por qué repetir el rango si ya hay `MinValue/MaxValue`? Porque `AllowOutOfRangeAutoCorrect`
corrige en el navegador, pero el POST puede llegar con cualquier número.

## RadMaskedTextBox

Propiedades: `Mask`, `PromptChar` (un solo carácter), `Text` (solo lo tecleado, sin
literales ni prompt), `TextWithLiterals` (lo tecleado + literales de la máscara; es lo que
usan los validators), `TextWithPrompt` y `TextWithPromptAndLiterals` (solo lectura).

Los símbolos de máscara (`#` = dígito, `L` = letra, etc.) **⚠ sin verificar**: consulta
la tabla de máscaras de tu versión.

```aspx
<telerik:RadMaskedTextBox ID="mtbTelefono" runat="server"
    Mask="(###) ###-####" PromptChar="_" />
<asp:RegularExpressionValidator ID="revTelefono" runat="server" ControlToValidate="mtbTelefono"
    ValidationExpression="^\(\d{3}\) \d{3}-\d{4}$"
    ErrorMessage="Captura un teléfono de 10 dígitos." Display="Dynamic" ValidationGroup="Alta" />
```

Servidor (guarda solo dígitos, usa `Text`):

```csharp
string tel = mtbTelefono.Text;            // "5512345678"
if (!Regex.IsMatch(tel, @"^\d{10}$"))      { /* "Captura un teléfono de 10 dígitos." */ }
```

Ojo: como el validator ve `TextWithLiterals`, la regex de cliente incluye paréntesis y
guion; la de servidor, aplicada a `Text`, no.

## RadDatePicker / RadDateInput

Propiedades de servidor: `MinDate`, `MaxDate` (por defecto 1/1/1980 y 12/31/2099),
`SelectedDate` (`DateTime?`), `DbSelectedDate` (igual que `SelectedDate` pero no lanza
excepción con `null`/`DBNull`; úsalo al enlazar con datos), `DateInput.DateFormat`
(formato al editar) y `DateInput.DisplayDateFormat` (formato sin foco).

API cliente: `get_selectedDate()` (devuelve `null` si no hay fecha), `set_minDate()`,
`set_maxDate()`, `get_dateInput()`; evento `ClientEvents-OnDateSelected`.

Una fecha fuera de `MinDate/MaxDate` asignada desde la base (p. ej. un "centinela"
1900-01-01) provoca un error de servidor. La propiedad `SkipMinMaxDateValidationOnServer`
existe en versiones recientes según un hilo del foro **⚠ sin verificar**.

### Patrón: dos pickers como rango (inicio limita el mínimo del fin)

```aspx
<telerik:RadDatePicker ID="dpInicio" runat="server"
    MinDate="2020-01-01" MaxDate="2030-12-31"
    DateInput-DateFormat="dd/MM/yyyy" DateInput-DisplayDateFormat="dd/MM/yyyy">
    <ClientEvents OnDateSelected="dpInicio_DateSelected" />
</telerik:RadDatePicker>

<telerik:RadDatePicker ID="dpFin" runat="server"
    MinDate="2020-01-01" MaxDate="2030-12-31"
    DateInput-DateFormat="dd/MM/yyyy" DateInput-DisplayDateFormat="dd/MM/yyyy" />

<asp:RequiredFieldValidator ID="rfvInicio" runat="server" ControlToValidate="dpInicio"
    ErrorMessage="La fecha inicio es obligatoria." Display="Dynamic" ValidationGroup="Alta" />
<asp:RequiredFieldValidator ID="rfvFin" runat="server" ControlToValidate="dpFin"
    ErrorMessage="La fecha fin es obligatoria." Display="Dynamic" ValidationGroup="Alta" />
<asp:CompareValidator ID="cvRango" runat="server"
    ControlToValidate="dpFin" ControlToCompare="dpInicio"
    Operator="GreaterThanEqual" Type="Date"
    ErrorMessage="La fecha fin debe ser igual o posterior a la fecha inicio."
    Display="Dynamic" ValidationGroup="Alta" />

<script type="text/javascript">
    function dpInicio_DateSelected(sender, args) {
        var fin = $find("<%= dpFin.ClientID %>");
        var inicio = sender.get_selectedDate();
        if (inicio) {
            fin.set_minDate(inicio);
            var actual = fin.get_selectedDate();
            if (actual && actual < inicio) { fin.clear(); }
        }
    }
</script>
```

- `<%= dpFin.ClientID %>` es seguro aquí: es un ID generado por el servidor, no un dato del
  usuario. La regla de `<%: %>` aplica a **datos**.
- `fin.clear()` **⚠ sin verificar** como nombre del método cliente para limpiar la selección.
- Un hilo del foro indica que, con la API antigua, cambiar el máximo del picker no
  actualizaba el rango del `DateInput` interno; si ves ese síntoma, ajusta también el
  `DateInput` **⚠ sin verificar** en la API actual.
- `CompareValidator` con `Type="Date"` funciona con RadDatePicker porque el valor validado
  del control es la fecha en formato invariable **⚠ sin verificar**; si en tu versión no
  compara bien, usa un `CustomValidator` (ver `aspnet-validators.md`).

Servidor (la regla real):

```csharp
DateTime? ini = dpInicio.SelectedDate, fin = dpFin.SelectedDate;
if (!ini.HasValue || !fin.HasValue) { /* ambas obligatorias */ }
else if (ini.Value.Date > fin.Value.Date) { /* "La fecha fin debe ser igual o posterior..." */ }
else if ((fin.Value - ini.Value).TotalDays > 366) { /* "El rango máximo es de un año." */ }
// Además: ini/fin dentro de los límites de negocio, no solo de MinDate/MaxDate.
```

## RadInputManager

Aplica reglas a controles `asp:TextBox` estándar sin cambiarlos por controles Rad.
Settings: `TextBoxSetting`, `NumericTextBoxSetting`, `DateInputSetting`,
`RegExpTextBoxSetting`, `MaskedTextBoxSetting`. Cada setting necesita al menos un
`TargetInput` en `TargetControls` (sin destino no se serializa al cliente).

- Requerido: `Validation.IsRequired` (la propiedad `IsRequiredFields` está **obsoleta**).
- Grupo: `Validation.ValidationGroup`.
- `RegExpTextBoxSetting`: `ValidationExpression`, `ErrorMessage`.
- `EmptyMessage` es un marcador; si el campo requerido queda vacío, el mensaje de error lo
  sustituye.
- Propiedades de rango de `NumericTextBoxSetting` / `DateInputSetting`
  (`MinValue`, `MaxValue`, `MinDate`, `MaxDate`) **⚠ sin verificar**.

```aspx
<asp:TextBox ID="txtCurp" runat="server" MaxLength="18" />

<telerik:RadInputManager ID="rim" runat="server">
    <telerik:RegExpTextBoxSetting BehaviorID="curpSetting"
        ValidationExpression="^[A-Z][AEIOUX][A-Z]{2}\d{6}[HM][A-Z]{5}[A-Z0-9]\d$"
        ErrorMessage="CURP inválida.">
        <Validation IsRequired="true" ValidationGroup="Alta" />
        <TargetControls>
            <telerik:TargetInput ControlID="txtCurp" />
        </TargetControls>
    </telerik:RegExpTextBoxSetting>
</telerik:RadInputManager>
```

La forma declarativa `<Validation IsRequired="..." />` y `BehaviorID` **⚠ sin verificar**
(los documentos confirman `Validation.IsRequired` como propiedad; en code-behind:
`setting.Validation.IsRequired = true;`).

Servidor (RadInputManager es solo cliente para estas reglas):

```csharp
string curp = txtCurp.Text.Trim().ToUpperInvariant();
if (curp.Length == 0) { /* obligatoria */ }
else if (!Regex.IsMatch(curp, PatronCurp)) { /* "CURP inválida." */ } // patrón completo en field-types.md
```

## RadComboBox

Por diseño, los validators validan el **Text** del RadComboBox, no el `Value`.

- Con un ítem "Seleccione…", usa `RequiredFieldValidator.InitialValue` igual al **texto**
  de ese ítem.
- Con `AllowCustomText="true"` y `EmptyMessage`, usa `InitialValue=""`.
- `SelectedItem` puede ser `null` con carga bajo demanda o `AllowCustomText`.

```aspx
<telerik:RadComboBox ID="cboEstado" runat="server" AllowCustomText="false">
    <Items>
        <telerik:RadComboBoxItem Text="Seleccione…" Value="" />
        <telerik:RadComboBoxItem Text="Jalisco" Value="14" />
    </Items>
</telerik:RadComboBox>
<asp:RequiredFieldValidator ID="rfvEstado" runat="server" ControlToValidate="cboEstado"
    InitialValue="Seleccione…" ErrorMessage="Selecciona un estado."
    Display="Dynamic" ValidationGroup="Alta" />
```

Servidor (el valor debe existir en el catálogo):

```csharp
if (!int.TryParse(cboEstado.SelectedValue, out int idEstado) || !catalogoEstados.ContainsKey(idEstado))
{ /* "Selecciona un estado válido." */ }
```

## RadAsyncUpload

Propiedades: `AllowedFileExtensions` (arreglo vacío = sin restricción), `MaxFileSize`
(bytes; `0` = sin límite), `TemporaryFolder` (por defecto `App_Data\RadUploadTemp`),
`TargetFolder` (los archivos válidos se mueven ahí automáticamente). Evento cliente
`OnClientValidationFailed` (args: `get_fileName()`, `get_row()`); evento servidor
`OnFileUploaded` con `e.File` (`ContentLength`, `GetExtension()`, `SaveAs()`) y `e.IsValid`.

Desde **2026.1.421** la validación CSRF de las peticiones de subida está activa por
defecto (cambio incompatible); en versiones anteriores no.

```aspx
<telerik:RadAsyncUpload ID="upDoc" runat="server"
    AllowedFileExtensions=".pdf,.jpg,.png" MaxFileSize="2097152"
    OnClientValidationFailed="upDoc_ValidationFailed"
    OnFileUploaded="upDoc_FileUploaded" />
<script type="text/javascript">
    function upDoc_ValidationFailed(sender, args) {
        alert("El archivo " + args.get_fileName() + " no es válido (solo PDF/JPG/PNG, máx. 2 MB).");
    }
</script>
```

La sintaxis `AllowedFileExtensions=".pdf,.jpg,.png"` (lista separada por comas en el
marcado) y `OnClientValidationFailed` como atributo directo **⚠ sin verificar**.

Servidor:

```csharp
static readonly HashSet<string> Permitidas =
    new HashSet<string>(StringComparer.OrdinalIgnoreCase) { ".pdf", ".jpg", ".png" };

protected void upDoc_FileUploaded(object sender, FileUploadedEventArgs e)
{
    bool ok = Permitidas.Contains(e.File.GetExtension())
              && e.File.ContentLength > 0
              && e.File.ContentLength <= 2 * 1024 * 1024;
    e.IsValid = ok;
    if (!ok) return;
    // Si NO usas TargetFolder, guarda tú con nombre generado:
    // e.File.SaveAs(Path.Combine(rutaSegura, Guid.NewGuid() + e.File.GetExtension()));
}
```

- No combines `TargetFolder` con `SaveAs()` propio: el control ya movió el archivo y
  obtendrás "file not found".
- La extensión la decide el cliente; si el archivo se procesa, revisa también el contenido.

## RadGrid (edición e inserción)

- `ValidationSettings-EnableValidation` (por defecto `true`) y
  `ValidationSettings-CommandsToValidate` (por defecto `PerformInsert, Update`).
- `ColumnValidationSettings` en `GridBoundColumn`, `GridNumericColumn`,
  `GridHTMLEditorColumn`, `GridDropDownColumn` y `GridDateTimeColumn` genera un
  `RequiredFieldValidator`: `EnableRequiredFieldValidation="true"`.
- Subpropiedad `RequiredFieldValidator-ErrorMessage` dentro de `ColumnValidationSettings`
  **⚠ sin verificar**.
- Para reglas propias: agrega un `CustomValidator` en `ItemCreated` cuando
  `e.Item.IsInEditMode`, o usa `GridTemplateColumn` con validators en `EditItemTemplate`.
- Da `ValidationGroup` distinto al formulario de inserción y al de edición si ambos
  pueden estar abiertos.

```aspx
<telerik:RadGrid ID="grdProductos" runat="server" AutoGenerateColumns="false"
    OnInsertCommand="grdProductos_InsertCommand" OnUpdateCommand="grdProductos_UpdateCommand">
    <ValidationSettings EnableValidation="true" CommandsToValidate="PerformInsert,Update" />
    <MasterTableView DataKeyNames="Id" CommandItemDisplay="Top">
        <Columns>
            <telerik:GridBoundColumn DataField="Nombre" HeaderText="Nombre" UniqueName="Nombre">
                <ColumnValidationSettings EnableRequiredFieldValidation="true">
                    <RequiredFieldValidator ErrorMessage="El nombre es obligatorio." />
                </ColumnValidationSettings>
            </telerik:GridBoundColumn>
            <telerik:GridTemplateColumn HeaderText="Precio" UniqueName="Precio">
                <ItemTemplate><%#: Eval("Precio", "{0:C}") %></ItemTemplate>
                <EditItemTemplate>
                    <telerik:RadNumericTextBox ID="ntbPrecio" runat="server"
                        MinValue="0.01" MaxValue="99999.99" DbValue='<%# Bind("Precio") %>' />
                    <asp:RequiredFieldValidator ID="rfvPrecio" runat="server"
                        ControlToValidate="ntbPrecio" ErrorMessage="El precio es obligatorio."
                        Display="Dynamic" />
                </EditItemTemplate>
            </telerik:GridTemplateColumn>
        </Columns>
    </MasterTableView>
</telerik:RadGrid>
```

`DbValue` en RadNumericTextBox (análogo a `DbSelectedDate`) **⚠ sin verificar**.

Servidor (en `InsertCommand`/`UpdateCommand`; `e.Canceled = true` cancela la operación):

```csharp
protected void grdProductos_UpdateCommand(object sender, GridCommandEventArgs e)
{
    var item = (GridEditableItem)e.Item;
    var valores = new Hashtable();
    item.ExtractValuesFromItem(valores, item);

    string nombre = (valores["Nombre"] as string ?? "").Trim();
    var ntb = (RadNumericTextBox)item.FindControl("ntbPrecio");

    if (!Page.IsValid || nombre.Length == 0 || nombre.Length > 100
        || !ntb.Value.HasValue || ntb.Value <= 0 || ntb.Value > 99999.99)
    {
        e.Canceled = true;   // el registro sigue en edición
        // muestra el error (Label o RadWindow) junto al formulario de edición
        return;
    }
    // UPDATE con SqlParameter (ver server-security.md)
}
```

Nota: si `ExtractValuesFromItem` incluye o no las columnas de plantilla depende de que
uses `Bind()`; por eso el ejemplo lee `ntbPrecio` con `FindControl`.

## RadEditor (contenido HTML)

RadEditor es la **excepción justificada** a "no aceptar HTML": su propósito es recibir
marcado. Por eso aquí sí hay filtrado, pero como capa adicional a la validación de servidor.

- Filtros de seguridad activos por defecto: `RemoveScripts`, `EncodeScripts`,
  `StripCssExpressions`, `StripDomEventAttributes` (activo por defecto desde 2025 Q4),
  `StripJavaScriptUris`.
- Se configuran con la propiedad `ContentFilters` o con `DisableFilter()` /
  `EnableFilter()`. `ContentFilters="None"` apaga **todos** los filtros: no lo hagas.
- Si `RemoveScripts` está activo, `EncodeScripts` no tiene efecto.
- Telerik indica que la seguridad sigue siendo responsabilidad del desarrollador: el POST
  se puede modificar sin pasar por el editor.

```aspx
<telerik:RadEditor ID="edDescripcion" runat="server"
    ContentFilters="DefaultFilters" MaxHtmlLength="20000" />
```

El valor de enumeración `DefaultFilters` y la propiedad `MaxHtmlLength` **⚠ sin verificar**.

Servidor:

```csharp
string html = edDescripcion.Content;
if (html.Length > 20000) { /* "La descripción es demasiado larga." */ }
// Sanitiza con lista blanca en servidor (p. ej. una librería tipo HtmlSanitizer)
// ANTES de guardar o, como mínimo, antes de pintar. No uses una regex anti-<script>.
string seguro = sanitizador.Sanitize(html);
```

La columna destino debe ser `NVARCHAR(MAX)` o del tamaño que valides; el HTML crece más que
el texto visible.

## RadButton

`CausesValidation`, `ValidationGroup` (vacío = valida todos los validators sin grupo),
`SingleClick` + `SingleClickText` (deshabilita el botón tras el clic para evitar dobles
envíos; solo en botones que hacen postback).

```aspx
<telerik:RadButton ID="btnGuardar" runat="server" Text="Guardar"
    CausesValidation="true" ValidationGroup="Alta"
    SingleClick="true" SingleClickText="Guardando…"
    OnClick="btnGuardar_Click" />
<telerik:RadButton ID="btnCancelar" runat="server" Text="Cancelar"
    CausesValidation="false" OnClick="btnCancelar_Click" />
```

Si la validación de cliente falla con `SingleClick="true"`, el botón puede quedarse
deshabilitado; la solución documentada en el foro es re-habilitarlo desde
`OnClientClicking` con `set_enabled(true)`.

Servidor:

```csharp
protected void btnGuardar_Click(object sender, EventArgs e)
{
    Page.Validate("Alta");          // explícito: no dependas de que el cliente validó
    if (!Page.IsValid) return;
    if (!ValidarReglasDeNegocio(out string error)) { lblError.Text = HttpUtility.HtmlEncode(error); return; }
    Guardar();
}
```

`SingleClick` es UX: el servidor debe tolerar doble envío (idempotencia o clave única).

---

## Fuentes

Consultadas mediante búsqueda restringida a dominios oficiales (no se pudo abrir cada
página directamente desde el entorno de redacción):

- Release history 2026 Q3 (2026.3.812): https://www.telerik.com/support/whats-new/aspnet-ajax/release-history/telerik-ui-for-asp-net-ajax-2026-3-812-(2026-q3)
- Release history 2026.2.708: https://www.telerik.com/support/whats-new/aspnet-ajax/release-history/telerik-ui-for-asp-net-ajax-2026-2-708
- RadDatePicker client object: https://docs.telerik.com/devtools/aspnet-ajax/controls/datepicker/client-side-programming/raddatepicker-object
- RadDatePicker client API: https://docs.telerik.com/devtools/aspnet-ajax/api/client/Telerik.Web.UI.RadDatePicker
- RadDatePicker OnDateSelected: https://docs.telerik.com/devtools/aspnet-ajax/controls/datepicker/client-side-programming/events/ondateselected
- RadDatePicker server API: https://docs.telerik.com/devtools/aspnet-ajax/api/server/Telerik.Web.UI/RadDatePicker
- RadDateInput getting and setting values: https://docs.telerik.com/devtools/aspnet-ajax/controls/dateinput/features/getting-and-setting-values
- RadDatePicker formatting values: https://docs.telerik.com/devtools/aspnet-ajax/controls/datepicker/functionality/formatting-values
- RadTextBox properties: https://docs.telerik.com/devtools/aspnet-ajax/controls/textbox/server-side-programming/properties
- RadTextBox validation: https://docs.telerik.com/devtools/aspnet-ajax/controls/textbox/features/validation
- RadNumericTextBox server API: https://www.telerik.com/products/aspnet-ajax/documentation/api/server/telerik.web.ui/radnumerictextbox
- RadNumericTextBox formatting: https://docs.telerik.com/devtools/aspnet-ajax/controls/numerictextbox/features/formatting-numeric-values
- RadMaskedTextBox getting and setting values: https://docs.telerik.com/devtools/aspnet-ajax/controls/maskedtextbox/features/getting-and-setting-values
- RadMaskedTextBox server API: https://docs.telerik.com/devtools/aspnet-ajax/api/server/Telerik.Web.UI/RadMaskedTextBox
- RadInputManager overview: https://docs.telerik.com/devtools/aspnet-ajax/controls/inputmanager/overview
- RegExpTextBoxSetting API: https://docs.telerik.com/devtools/aspnet-ajax/api/server/Telerik.Web.UI/RegExpTextBoxSetting
- RadComboBox validate by value: https://docs.telerik.com/devtools/aspnet-ajax/controls/combobox/how-to/validate-combobox-by-value
- RadComboBox default item: https://docs.telerik.com/devtools/aspnet-ajax/controls/combobox/radcombobox-items/default-item
- RadAsyncUpload server API: https://docs.telerik.com/devtools/aspnet-ajax/api/server/Telerik.Web.UI/RadAsyncUpload
- RadAsyncUpload OnClientValidationFailed: https://docs.telerik.com/devtools/aspnet-ajax/controls/asyncupload/client-side-programming/onclientvalidationfailed
- RadAsyncUpload OnFileUploaded: https://docs.telerik.com/devtools/aspnet-ajax/controls/asyncupload/server-side-programming/onfileuploaded
- RadAsyncUpload manipulate uploaded files: https://docs.telerik.com/devtools/aspnet-ajax/controls/asyncupload/how-to/how-to-manipulate-the-uploaded-files
- RadGrid validation: https://docs.telerik.com/devtools/aspnet-ajax/controls/grid/data-editing/validation
- RadGrid InsertCommand: https://docs.telerik.com/devtools/aspnet-ajax/controls/grid/server-side-programming/events/insertcommand
- RadGrid updating values: https://docs.telerik.com/devtools/aspnet-ajax/controls/grid/data-editing/update-records/updating-values-using-inplace-and-editforms-modes
- RadEditor prevent XSS: https://docs.telerik.com/devtools/aspnet-ajax/controls/editor/managing-content/prevent-cross-site-scripting-(xss)
- RadEditor sanitize on paste (KB): https://www.telerik.com/products/aspnet-ajax/documentation/knowledge-base/editor-sanitize-html-content
- RadButton server API: https://docs.telerik.com/devtools/aspnet-ajax/api/server/Telerik.Web.UI/RadButton
- RadButton single click: https://docs.telerik.com/devtools/aspnet-ajax/controls/button/functionality/single-click-button
- Foro: re-habilitar RadButton con SingleClick: https://www.telerik.com/forums/re-enable-radbutton-with-property-singleclick=true-without-postback
