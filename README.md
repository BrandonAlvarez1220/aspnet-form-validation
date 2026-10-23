# aspnet-form-validation

Plugin de [Claude Code](https://code.claude.com/docs) con una skill (`form-validation`)
que guía a Claude al **crear o revisar formularios ASP.NET Web Forms (ASPX)** con
Telerik UI for ASP.NET AJAX, Kendo UI o HTML5 plano, con code-behind en C# y SQL Server.

## Qué hace

Cuando trabajas en un formulario, la skill hace que Claude aplique un checklist por campo
(requerido, tipo, longitud, rango y formato) en **tres capas**:

1. **Cliente**: UX con validators de ASP.NET, propiedades de controles Telerik/Kendo o
   atributos HTML5.
2. **Servidor**: la validación real. Siempre se repite en C# (`Page.IsValid`, reglas en
   code-behind y `SqlParameter`).
3. **Salida**: codificación al pintar (`<%: %>`, `<%#: %>`, `HttpUtility.HtmlEncode`) contra XSS.

También alinea el `MaxLength` de cada control con el tamaño de su columna en SQL Server y
mantiene los mensajes de error consistentes, en español y junto al campo.

## Por qué existe

En Web Forms es fácil confiar en lo que se ve en el navegador: un `MaxLength`, un
`RadDatePicker` con `MaxDate` o un `required`. Ninguno de ellos protege nada, porque una
petición se puede fabricar sin navegador. Esta skill hace que la regla "ninguna validación
existe solo en el cliente" se cumpla cada vez que Claude escribe o revisa un formulario.

## Instalación

Dentro de Claude Code:

```
/plugin marketplace add BrandonAlvarez1220/aspnet-form-validation
/plugin install aspnet-form-validation@aspnet-form-validation
```

Desde el shell:

```bash
claude plugin marketplace add BrandonAlvarez1220/aspnet-form-validation
claude plugin install aspnet-form-validation@aspnet-form-validation
```

Para probarlo localmente desde un clon del repositorio:

```bash
claude plugin validate .
claude plugin marketplace add ./
claude plugin install aspnet-form-validation@aspnet-form-validation
```

La skill se activa sola cuando la tarea coincide con su descripción (formularios ASPX,
Telerik, Kendo, HTML5). También puedes invocarla con `/aspnet-form-validation:form-validation`.

## Estructura

```
.
├── .claude-plugin/
│   ├── plugin.json              # manifiesto del plugin
│   └── marketplace.json         # este repo es también su propio marketplace
├── skills/
│   └── form-validation/
│       ├── SKILL.md             # reglas cortas + qué referencia leer en cada caso
│       └── references/
│           ├── field-types.md       # regla por tipo de campo (texto, número, RFC, CURP, fechas…)
│           ├── telerik-aspx.md      # RadTextBox, RadDatePicker, RadGrid, RadEditor, RadButton…
│           ├── aspnet-validators.md # Required/Range/RegularExpression/Compare/CustomValidator
│           ├── server-security.md   # Page.IsValid, SqlParameter, codificación, ViewState
│           ├── html5.md             # required, pattern, maxlength… (solo UX)
│           └── kendo.md             # DatePicker, NumericTextBox, Validator
└── README.md
```

`SKILL.md` es corto a propósito: Claude solo carga el archivo de `references/` que
necesita el caso, lo que ahorra contexto.

## Limitaciones (nota honesta)

- **La skill no valida nada en tiempo de ejecución.** Son instrucciones que guían a Claude
  al escribir o revisar código; no es una librería ni se instala en tu aplicación. La
  validación real es el código que acabe en tu proyecto, y lo tienes que revisar y probar tú.
- **Algunos nombres de API están marcados "⚠ sin verificar".** Los nombres de propiedades,
  eventos y clases de Telerik, Kendo y ASP.NET se contrastaron con la documentación oficial
  (docs.telerik.com, telerik.com, learn.microsoft.com). Lo que no se pudo confirmar se dejó
  marcado en el archivo para que lo compruebes contra tu versión antes de usarlo. Cada
  referencia de Telerik y Kendo termina con una sección "Fuentes".
- `telerik-aspx.md` se redactó contra la documentación más reciente de Telerik UI for
  ASP.NET AJAX (2026 Q3) y no contra una versión concreta. Si usas una versión anterior,
  revisa las notas de versión.
- Los patrones de RFC, CURP, teléfono y CP son de México y validan forma, no existencia.
