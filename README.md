# Informe de Limpieza y Transformación de Datos de Contactos

Este documento detalla el proceso realizado para limpiar y transformar datos de **Contactos de Clientes**, **Empleados** y **Proveedores**. Los pasos están orientados a la preparación de datos para análisis, integridad y estandarización.

---

## Contactos Clientes

**Paso 1:** Cargar los datos originales.

**Paso 2:** Transformar los datos.
- Usar la primera fila como encabezado de las columnas (los nombres de las columnas).

**Paso 3:** Convertir todos los correos electrónicos a minúsculas.

**Paso 4:** Eliminar los guiones (`-`) en los nombres.

**Paso 5:** Corregir los correos que tienen doble punto (`..`) al final como `.com..`, reemplazando por un solo punto (`.com`).

**Paso 6:** Limpiar los datos de teléfono creando una columna personalizada con el siguiente comando:
```m
= if Text.StartsWith([Telefono], "+") and Text.Length(Text.Select([Telefono], {"0".."9"})) > 6
    then [Telefono]
else if Text.StartsWith([Pais], "+") and Text.Length(Text.Select([Pais], {"0".."9"})) > 6
    then [Pais]
else null
```

**Paso 7:** Limpiar los datos del país creando una columna personalizada y quitando los números telefónicos, usando:
```m
let
    paisesValidos = {"Colombia", "México", "Perú", "Chile", "Argentina", "Ecuador", "Venezuela", "Guatemala"}
in
    if List.Contains(paisesValidos, [Pais]) then [Pais]
    else if List.Contains(paisesValidos, [Telefono]) then [Telefono]
    else if List.Contains(paisesValidos, [#"País 2"]) then [#"País 2"]
    else null
```

**Paso 8:** Establecer en `null` los valores de la columna `pais_limpio` que contienen errores o números telefónicos.

**Paso 9:** Limpiar la columna de correo y agregar correctamente la finalización de los correos (dominios), usando:
```m
let
    correo = Text.Trim([Correo]),
    telefono = Text.Trim([Telefono]),

    correoValido =
        Text.Contains(correo, "@") and 
        Text.PositionOf(Text.Range(correo, Text.PositionOf(correo, "@")), ".") > 0,

    esDominioCorto =
        Text.Length(telefono) >= 2 and Text.Length(telefono) <= 4 and
        not Text.Contains(telefono, " ") and
        not Text.Contains(telefono, "+") and
        not Text.Contains(telefono, "-") and
        not Text.Contains(telefono, "@") and
        not Text.Contains(telefono, "/"),

    correoCorregido = 
        if not correoValido and esDominioCorto and correo <> null and correo <> "" 
            then correo & "." & telefono
        else correo,

    correoFinal = 
        if Text.Contains(correoCorregido, "@") and Text.PositionOf(Text.Range(correoCorregido, Text.PositionOf(correoCorregido, "@")), ".") > 0
            then Text.Lower(correoCorregido)
        else null
in
    correoFinal
```

**Paso 10:** Nombrar la columna que contiene algunos nombres de países como `PaisExtra` y combinarla con `Pais_limpio` usando:
```m
if [Pais_limpio] <> null and [Pais_limpio] <> "" then [Pais_limpio]
else if [PaisExtra] <> null and [PaisExtra] <> "" then [PaisExtra]
else null
```

**Paso 11:** Eliminar columnas innecesarias y renombrar las columnas finales como Teléfono, País, Correo, etc.

**Paso 12:** Guardar la tabla resultante y nombrarla como `Contactos_clientes_final`.

---

## Contactos Empleados

**Paso 1:** Asignar la primera fila como encabezado de las columnas.

**Paso 2:** Cambiar el valor `na` de la columna nombre a `null`.

**Paso 3:** Quitar doble espaciado y guiones (`-`) en los nombres.

**Paso 4:** Cambiar el valor `n/a` y los campos vacíos a `null`.

**Paso 5:** Renombrar la columna `e-mail` a `correo` para evitar errores, y usar:
```m
let
    correo = Text.Trim([e-mail]),
    celular = Text.Trim([Celular]),

    correoValido =
        Text.Contains(correo, "@") and 
        Text.PositionOf(Text.Range(correo, Text.PositionOf(correo, "@")), ".") > 0,

    esDominioCorto =
        Text.Length(celular) >= 2 and Text.Length(celular) <= 4 and
        List.NonNullCount(
            List.Transform(
                Text.ToList(celular),
                (c) => if c >= "A" and c <= "Z" or c >= "a" and c <= "z" then 1 else null
            )
        ) = Text.Length(celular),

    correoCorregido = 
        if not correoValido and esDominioCorto and correo <> null and correo <> "" 
            then correo & "." & celular
        else correo,

    correoFinal = 
        if Text.Contains(correoCorregido, "@") and Text.PositionOf(Text.Range(correoCorregido, Text.PositionOf(correoCorregido, "@")), ".") > 0
            then Text.Lower(correoCorregido)
        else correoCorregido
in
    try correoFinal otherwise null
```

**Paso 6:** Cambiar la columna resultante (`correo_limpio`) a tipo texto.

**Paso 7:** Limpiar y unificar números telefónicos de las columnas `Celular` y `Pais`, usando:
```m
let
    celular = Text.Trim([Celular]),
    pais = Text.Trim([Pais]),

    limpiarNumero = (txt as text) as text =>
        Text.Combine(
            List.Select(
                Text.ToList(txt),
                (c) => (c >= "0" and c <= "9") or c = "+"
            ),
            ""
        ),

    telefonoCelular = limpiarNumero(celular),
    telefonoPais = limpiarNumero(pais),

    telefonoFinal =
        if Text.Length(telefonoCelular) >= 7 then telefonoCelular
        else if Text.Length(telefonoPais) >= 7 then telefonoPais
        else null
in
    telefonoFinal
```

**Paso 8:** Limpiar columna país y extraer valores correctos de columnas sin nombre, usando:
```m
let
    valorPais = Text.Trim([Pais]),
    valorExtra = Text.Trim([PaisExtra]),

    esTelefono = 
        let
            soloNumeros = Text.Combine(
                List.Select(
                    Text.ToList(valorPais),
                    (c) => (c >= "0" and c <= "9")
                ),
                ""
            )
        in
            Text.Length(soloNumeros) >= 7,

    Pais_limpio = if esTelefono then if valorExtra <> null and valorExtra <> "" then valorExtra else null else valorPais,

    PaisExtra_limpio = if esTelefono then valorPais else if valorExtra <> null and valorExtra <> "" then valorExtra else null
in
    [Pais_limpio = Pais_limpio, PaisExtra_limpio = PaisExtra_limpio]
```

**Paso 9:** Eliminar columnas no útiles y renombrar las finales a `e-mail`, `celular`, `pais`, etc.

**Paso 10:** Guardar la tabla final como `contactos_empleados_final`.

---

## Contactos Proveedores

**Paso 1:** Convertir la primera fila en encabezado de columnas.

**Paso 2:** Limpiar la columna `Email` y agregar el dominio de la columna `Tel`, usando:
```m
let
    correo = Text.Trim([Email]),
    tel = Text.Trim([Tel]),

    correoValido =
        Text.Contains(correo, "@") and 
        Text.PositionOf(Text.Range(correo, Text.PositionOf(correo, "@")), ".") > 0,

    esDominioCorto =
        Text.Length(tel) >= 2 and Text.Length(tel) <= 4 and
        List.NonNullCount(
            List.Transform(
                Text.ToList(tel),
                (c) => if c >= "A" and c <= "Z" or c >= "a" and c <= "z" then 1 else null
            )
        ) = Text.Length(tel),

    correoCorregido = 
        if not correoValido and esDominioCorto and correo <> null and correo <> "" 
            then correo & "." & tel
        else correo,

    correoFinal = 
        if Text.Contains(correoCorregido, "@") and Text.PositionOf(Text.Range(correoCorregido, Text.PositionOf(correoCorregido, "@")), ".") > 0
            then Text.Lower(correoCorregido)
        else correoCorregido
in
    correoFinal
```

**Paso 3:** Limpiar la columna `Tel`, trayendo los teléfonos faltantes de `Country` y quitando valores no útiles:
```m
let
    tel = Text.Trim([Tel]),
    country = Text.Trim([Country]),

    limpiarNumero = (txt as text) as text =>
        Text.Combine(
            List.Select(
                Text.ToList(txt),
                (c) => (c >= "0" and c <= "9") or c = "+"
            ),
            ""
        ),

    telLimpio = limpiarNumero(tel),
    countryLimpio = limpiarNumero(country),

    resultado =
        if Text.Length(telLimpio) >= 7 then telLimpio
        else if Text.Length(countryLimpio) >= 7 then countryLimpio
        else null
in
    resultado
```

**Paso 4:** Nombrar la columna vacía como `CountryExtra`.

**Paso 5:** Limpiar la columna `Country`, eliminando teléfonos y combinando con `CountryExtra`:
```m
let
    valCountry = Text.Trim([Country]),
    valCountryExtra = try Text.Trim([CountryExtra]) otherwise null,

    codigosPais = {"CO", "MX", "CL", "PE", "VE", "AR", "BO", "EC", "GT"},

    soloNumeros = Text.Combine(
        List.Select(
            Text.ToList(valCountry),
            (c) => (c >= "0" and c <= "9")
        ),
        ""
    ),
    esTelefono = Text.Length(soloNumeros) >= 7,

    resultado = 
        if esTelefono or valCountry = null or valCountry = "" then
            if valCountryExtra <> null and List.Contains(codigosPais, Text.Upper(valCountryExtra)) then Text.Upper(valCountryExtra) else null
        else if List.Contains(codigosPais, Text.Upper(valCountry)) then Text.Upper(valCountry)
        else null
in
    resultado
```

**Paso 6:** Quitar columnas innecesarias y renombrar las columnas finales como `Email`, `Tel`, `Country`.

**Paso 7:** Guardar la tabla final como `contactos_proveedores_final`.

---

## Resultados

- Los datos finales están estandarizados, limpios y listos para análisis y uso posterior.
- Se recomienda revisar la lógica aplicada en cada consulta personalizada para adaptarla a posibles cambios futuros en la estructura de los datos.

---
**Autor:** YOUNGDANY11  
**Fecha del informe:** 2025-10-01