# Validators estándar de ASP.NET Web Forms

Todos derivan de `BaseValidator` y comparten:

- `ControlToValidate`, `ErrorMessage` (texto para `ValidationSummary`), `Text` (texto junto
  al campo; si se omite se usa `ErrorMessage`).
- `Display`: `Static` | `Dynamic` | `None`. `Dynamic` solo ocupa espacio cuando hay error
  (y solo aplica con validación de cliente activa).
- `ValidationGroup`, `EnableClientScript`, `SetFocusOnError` (por defecto `false`; si
  varios fallan, recibe el foco el primero).

**Corren en cliente y en servidor.** El servidor los vuelve a ejecutar en el postback
(`Page.Validate`), por eso son la forma más barata de tener la misma regla en ambas capas.
Aun así, comprueba `Page.IsValid` antes de procesar (ver `server-security.md`).

> En .NET 4.5+ con `ValidationSettings:UnobtrusiveValidationMode = WebForms` en
> `appSettings`, los validators requieren un `ScriptResourceMapping` llamado `jquery`
> (sensible a mayúsculas). Si aparece el error "UnobtrusiveValidationMode requires a
> ScriptResourceMapping for 'jquery'", regístralo en `Application_Start` o usa `None`.

## RequiredFieldValidator

```aspx
<asp:TextBox ID="txtCorreo" runat="server" MaxLength="254" />
<asp:RequiredFieldValidator ID="rfvCorreo" runat="server" ControlToValidate="txtCorreo"
    ErrorMessage="El correo es obligatorio." Display="Dynamic" SetFocusOnError="true"
    ValidationGroup="Alta" />
```

- Es el **único** validator que falla con el campo vacío. Los demás (`Range`, `Compare`,
  `RegularExpression`) **pasan** si el campo está vacío; combínalos con este.
- `InitialValue`: valor que se considera "vacío" (p. ej. el texto de "Seleccione…").

## RangeValidator

```aspx
<asp:RangeValidator ID="rvEdad" runat="server" ControlToValidate="txtEdad"
    Type="Integer" MinimumValue="18" MaximumValue="99"
    ErrorMessage="La edad debe estar entre 18 y 99." Display="Dynamic" ValidationGroup="Alta" />
```

- `Type`: `String`, `Integer`, `Double`, `Date`, `Currency`.
- `MinimumValue` y `MaximumValue` deben poder convertirse a `Type`, si no, el control
  lanza excepción (también si quedan vacíos).
- Para fechas usa formato neutral (`yyyy/MM/dd`) en los límites.
- Fechas dinámicas (p. ej. "hoy") se asignan en code-behind:
  `rvFecha.MaximumValue = DateTime.Today.ToString("yyyy/MM/dd");`

## RegularExpressionValidator

```aspx
<asp:RegularExpressionValidator ID="revCorreo" runat="server" ControlToValidate="txtCorreo"
    ValidationExpression="^[^@\s]+@[^@\s]+\.[^@\s]+$"
    ErrorMessage="El correo no tiene un formato válido." Display="Dynamic" ValidationGroup="Alta" />
```

- En cliente se evalúa con el motor de JavaScript y en servidor con `System.Text.RegularExpressions.Regex`.
  Usa la sintaxis común (JavaScript es un subconjunto). `(?i)` funciona solo en servidor.
- Para no distinguir mayúsculas, incluye ambos rangos (`[A-Za-z]`) o normaliza y valida
  con `CustomValidator`.
- Evita patrones con retroceso catastrófico (`(a+)+`): el mismo patrón corre en servidor
  (ReDoS). En código propio usa `Regex` con `matchTimeout`.

## CompareValidator

Tres usos:

1. **Contra otro control** (`ControlToCompare`) — p. ej. confirmar contraseña o rango de fechas.
2. **Contra un valor fijo** (`ValueToCompare`).
3. **Solo verificar tipo**: `Operator="DataTypeCheck"` (ignora `ControlToCompare`/`ValueToCompare`).

`Operator`: `Equal`, `NotEqual`, `GreaterThan`, `GreaterThanEqual`, `LessThan`,
`LessThanEqual`, `DataTypeCheck`. `Type`: igual que `RangeValidator`.
`CultureInvariantValues` indica si los valores se convierten a formato neutral antes de comparar.

### Ejemplo: rango de fechas

```aspx
<asp:TextBox ID="txtInicio" runat="server" TextMode="Date" />
<asp:TextBox ID="txtFin" runat="server" TextMode="Date" />

<asp:CompareValidator ID="cvInicioTipo" runat="server" ControlToValidate="txtInicio"
    Operator="DataTypeCheck" Type="Date"
    ErrorMessage="La fecha inicio no es válida." Display="Dynamic" ValidationGroup="Alta" />
<asp:CompareValidator ID="cvRango" runat="server"
    ControlToValidate="txtFin" ControlToCompare="txtInicio"
    Operator="GreaterThanEqual" Type="Date"
    ErrorMessage="La fecha fin debe ser igual o posterior a la fecha inicio."
    Display="Dynamic" ValidationGroup="Alta" />
```

- El formato que acepta `Type="Date"` depende de la cultura de la página
  (`Culture`/`UICulture` en `@Page` o `web.config`) salvo que uses `CultureInvariantValues`.
  Con `TextMode="Date"` el navegador envía `yyyy-MM-dd`: verifica que tu cultura lo acepte
  o usa `CustomValidator`.
- Con un calendario no gregoriano, `Type="Date"` solo valida en servidor.

## CustomValidator (cliente + servidor)

Para reglas que los otros no cubren (rango máximo de días, catálogo, reglas cruzadas).

```aspx
<asp:CustomValidator ID="cvDuracion" runat="server" ControlToValidate="txtFin"
    ClientValidationFunction="validarDuracion" OnServerValidate="cvDuracion_ServerValidate"
    ValidateEmptyText="false"
    ErrorMessage="El rango no puede exceder 31 días." Display="Dynamic" ValidationGroup="Alta" />

<script type="text/javascript">
    function validarDuracion(source, args) {
        var ini = new Date(document.getElementById("<%= txtInicio.ClientID %>").value);
        var fin = new Date(args.Value);
        args.IsValid = !isNaN(ini) && !isNaN(fin) && (fin - ini) / 86400000 <= 31;
    }
</script>
```

```csharp
protected void cvDuracion_ServerValidate(object source, ServerValidateEventArgs args)
{
    args.IsValid =
        DateTime.TryParseExact(txtInicio.Text, "yyyy-MM-dd", CultureInfo.InvariantCulture, DateTimeStyles.None, out var ini) &&
        DateTime.TryParseExact(args.Value,     "yyyy-MM-dd", CultureInfo.InvariantCulture, DateTimeStyles.None, out var fin) &&
        (fin - ini).TotalDays <= 31;
}
```

- `args.Value` / `args.IsValid` existen en cliente y servidor.
- **Siempre** implementa `OnServerValidate`. Una función de cliente sin su par de servidor
  se salta deshabilitando JavaScript.
- Sin `ControlToValidate`, `args.Value` llega vacío (útil para validar CheckBox o varias
  entradas a la vez).
- Por defecto no se ejecuta con el campo vacío; `ValidateEmptyText="true"` lo fuerza
  (comportamiento descrito en el foro de Microsoft; compruébalo en tu versión).

## ValidationSummary

```aspx
<asp:ValidationSummary ID="vsAlta" runat="server" ValidationGroup="Alta"
    HeaderText="Corrige lo siguiente:" DisplayMode="BulletList"
    ShowSummary="true" ShowMessageBox="false" />
```

- `DisplayMode`: `BulletList` (por defecto), `List`, `SingleParagraph`.
- `ShowMessageBox` solo funciona con script de cliente.
- `HeaderText` **no** se codifica: no pongas datos del usuario ahí.
- Cada `ValidationGroup` necesita su propio `ValidationSummary`.
- Patrón recomendado: `Text="*"` (o el mensaje corto) junto al campo y `ErrorMessage`
  completo en el resumen.

## ValidationGroup

Separa formularios independientes de la misma página (búsqueda vs. alta, inserción vs.
edición en un grid).

```aspx
<asp:TextBox ID="txtBuscar" runat="server" />
<asp:RequiredFieldValidator ID="rfvBuscar" runat="server" ControlToValidate="txtBuscar"
    ErrorMessage="Escribe algo para buscar." ValidationGroup="Buscar" />
<asp:Button ID="btnBuscar" runat="server" Text="Buscar" ValidationGroup="Buscar" OnClick="btnBuscar_Click" />

<asp:Button ID="btnGuardar" runat="server" Text="Guardar" ValidationGroup="Alta" OnClick="btnGuardar_Click" />
```

```csharp
protected void btnGuardar_Click(object sender, EventArgs e)
{
    Page.Validate("Alta");   // valida solo ese grupo
    if (!Page.IsValid) return;
    // ...
}
```

- `Page.Validate()` sin parámetros **ignora los grupos** y valida todo.
- `Page.IsValid` considera los validators del grupo validado; si lo lees en `Page_Load`,
  llama antes a `Page.Validate(...)`.
- Botones que no deben validar (Cancelar, Limpiar): `CausesValidation="false"`.

## Fuentes

- CompareValidator: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.comparevalidator?view=netframework-4.8.1
- CompareValidator.Operator: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.comparevalidator.operator?view=netframework-4.8.1
- BaseCompareValidator: https://learn.microsoft.com/es-es/dotnet/api/system.web.ui.webcontrols.basecomparevalidator?view=netframework-4.8.1
- RangeValidator: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.rangevalidator?view=netframework-4.8.1
- RegularExpressionValidator: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.regularexpressionvalidator?view=netframework-4.8.1
- Client-Side Validation for ASP.NET Server Controls: https://learn.microsoft.com/en-us/previous-versions/aspnet/yb52a4x0(v=vs.100)
- ReDoS (MSDN Magazine): https://learn.microsoft.com/en-us/archive/msdn-magazine/2010/may/security-briefs-regular-expression-denial-of-service-attacks-and-defenses
- CustomValidator: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.customvalidator?view=netframework-4.8.1
- ServerValidateEventArgs: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.servervalidateeventargs?view=netframework-4.8.1
- Foro (ValidateEmptyText): https://learn.microsoft.com/en-us/archive/msdn-technet-forums/9530284f-7cc9-4a77-aaaf-f836c69e61b9
- ValidationSummary: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.validationsummary?view=netframework-4.8.1
- ValidationSummary.HeaderText: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.validationsummary.headertext?view=netframework-4.8.1
- BaseValidator.Display: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.basevalidator.display?view=netframework-4.8
- BaseValidator.SetFocusOnError: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.basevalidator.setfocusonerror?view=netframework-4.8.1
- Page.Validate: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.page.validate?view=netframework-4.8.1
- Page.IsValid: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.page.isvalid?view=netframework-4.8.1
- UnobtrusiveValidationMode: https://learn.microsoft.com/en-us/dotnet/api/system.web.ui.unobtrusivevalidationmode?view=netframework-4.8.1
