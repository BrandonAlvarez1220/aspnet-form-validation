# HTML5: validación nativa del navegador

> **Solo es UX.** Los atributos HTML5 se eliminan con DevTools en dos segundos y no
> existen para quien envía la petición con curl. Cada regla de este archivo debe estar
> repetida en el servidor (`server-security.md`).

## Atributos

| Atributo | Efecto | Equivalente en servidor |
|---|---|---|
| `required` | No envía el formulario si está vacío | `!string.IsNullOrWhiteSpace(v)` |
| `maxlength="n"` | No deja escribir más de `n` caracteres (unidades UTF-16) | `v.Length <= n` = tamaño de la columna |
| `minlength="n"` | Exige al menos `n` caracteres (solo si el campo no está vacío) | `v.Length >= n` |
| `pattern="..."` | Regex que debe cubrir **todo** el valor (anclada implícitamente) | `Regex.IsMatch(v, "^...$")` |
| `type="email"` | Formato de correo básico | Regex de `field-types.md` |
| `type="number"` + `min`/`max`/`step` | Rango y granularidad | `TryParse` + rango |
| `type="date"` + `min`/`max` | Rango de fechas; el valor siempre viaja como `yyyy-MM-dd` | `TryParseExact(v, "yyyy-MM-dd", CultureInfo.InvariantCulture, ...)` |
| `title` | Texto de ayuda que muchos navegadores muestran junto al error de `pattern` | — |

## Ejemplo

```html
<form method="post" novalidate>  <!-- quita novalidate si quieres la burbuja nativa -->
  <label for="correo">Correo</label>
  <input id="correo" name="correo" type="email" required maxlength="254">

  <label for="cp">Código postal</label>
  <input id="cp" name="cp" required inputmode="numeric"
         pattern="\d{5}" maxlength="5" title="5 dígitos">

  <label for="inicio">Inicio</label>
  <input id="inicio" name="inicio" type="date" min="2020-01-01" max="2030-12-31" required>
  <label for="fin">Fin</label>
  <input id="fin" name="fin" type="date" min="2020-01-01" max="2030-12-31" required>
</form>

<script>
  // Rango: el inicio limita el mínimo del fin
  const ini = document.getElementById('inicio'), fin = document.getElementById('fin');
  ini.addEventListener('change', () => {
    fin.min = ini.value || '2020-01-01';
    if (fin.value && fin.value < ini.value) fin.value = '';
  });
</script>
```

## Mensajes en español

Los mensajes nativos salen en el idioma del navegador, no del sitio. Para mensajes
consistentes:

```js
const cp = document.getElementById('cp');
cp.addEventListener('invalid', () => {
  if (cp.validity.valueMissing)        cp.setCustomValidity('El código postal es obligatorio.');
  else if (cp.validity.patternMismatch) cp.setCustomValidity('El código postal debe tener 5 dígitos.');
});
cp.addEventListener('input', () => cp.setCustomValidity(''));  // limpia al corregir
```

O usa `novalidate` y pinta tus propios mensajes junto al campo con la Constraint
Validation API (`input.checkValidity()`, `input.validity`).

## Con controles ASP.NET

- `asp:TextBox` con `TextMode="Email"`, `TextMode="Number"`, `TextMode="Date"` genera el
  `type` correspondiente (ASP.NET 4.5+).
- Atributos que el control no expone (`pattern`, `required`, `min`) se agregan como
  atributos expandidos en el marcado (`<asp:TextBox ... pattern="\d{5}" />`) o con
  `txt.Attributes["pattern"] = ...`.
- Mezclar validación nativa con validators de ASP.NET puede mostrar dos mensajes; elige uno
  para la UX (normalmente los validators) y usa `novalidate` en el `<form>` si hace falta.

## Trampas

- `pattern` se ignora si el campo está vacío: combina con `required`.
- `maxlength` no se aplica a valores asignados por script ni a lo que llegue por POST.
- `type="number"` envía cadena vacía si el usuario teclea algo no numérico.
- `pattern` usa la sintaxis de JavaScript (con bandera `v` en navegadores recientes):
  escapa los caracteres especiales dentro de clases (`[\-]`).
