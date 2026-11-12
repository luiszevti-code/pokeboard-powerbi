# Modelo y DAX

Todo lo que se construye dentro del modelo de Power BI (después de cargar los datos desde Power Query): tablas calculadas, columnas, medidas y relaciones. Se hace en DAX y no en Power Query para no volver a llamar a la API.

**Orden de creación:** `DimGeneracion` → columnas de `Pokedex` (`Total`, `Clase`, `Tipos`) → `Linea` → `PokemonStats` → medidas.

## Relaciones

| Desde (uno) | Hacia (muchos) | Dirección del filtro |
|-------------|----------------|----------------------|
| `DimTipos[tipo]` | `PokemonTipos[tipo]` | Única |
| `Pokedex[id]` | `PokemonTipos[id]` | Ambas |
| `DimTipos[tipo]` | `EfectividadTipos[atacante]` | Única |
| `DimGeneracion[generacion]` | `Pokedex[generacion]` | Única |
| `Pokedex[cadena_id]` | `Linea[cadena_id]` (varios a varios) | Única, de `Pokedex` hacia `Linea` |
| `Pokedex[id]` | `PokemonStats[id]` | Única |

**Decisiones de diseño**
- `Pokedex` → `PokemonTipos` va en ambas direcciones para que un filtro por tipo llegue hasta los Pokémon. No genera ambigüedad en este modelo.
- Un Pokémon de dos tipos cuenta en ambos (por ejemplo, Charizard aparece en Fuego y en Volador).
- La relación varios a varios con `Linea` permite mostrar la línea evolutiva completa del Pokémon seleccionado, incluidas las evoluciones fuera del alcance. En los visuales de la ficha no se mezclan campos de `Pokedex` y de `Linea`, para evitar combinaciones duplicadas.

## Tablas calculadas

### DimGeneracion

```
DimGeneracion =
DATATABLE(
    "generacion", INTEGER,
    "region", STRING,
    "etiqueta", STRING,
    {
        {1, "Kanto", "Gen 1 · Kanto"},
        {2, "Johto", "Gen 2 · Johto"},
        {3, "Hoenn", "Gen 3 · Hoenn"},
        {4, "Sinnoh", "Gen 4 · Sinnoh"},
        {5, "Teselia", "Gen 5 · Teselia"},
        {6, "Kalos", "Gen 6 · Kalos"},
        {7, "Alola", "Gen 7 · Alola"},
        {8, "Galar", "Gen 8 · Galar"},
        {9, "Paldea", "Gen 9 · Paldea"}
    }
)
```

La columna `etiqueta` se ordena por `generacion`.

### Linea
Copia ligera de `Pokedex` para mostrar la línea evolutiva del Pokémon seleccionado.

```
Linea =
SELECTCOLUMNS(
    Pokedex,
    "cadena_id", Pokedex[cadena_id],
    "linea_id", Pokedex[id],
    "linea_nombre", Pokedex[nombre],
    "linea_imagen", Pokedex[imagen_url],
    "linea_etapa", Pokedex[etapa],
    "linea_tipo_1", Pokedex[tipo_1],
    "linea_tipo_2", Pokedex[tipo_2],
    "linea_total", Pokedex[Total]
)
```

`linea_imagen` lleva la categoría de datos *URL de imagen*.

### PokemonStats
Estadísticas en formato largo (una fila por Pokémon y estadística), para los gráficos de barras.

```
PokemonStats =
UNION(
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "HP", "orden", 1, "valor", Pokedex[hp]),
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "Ataque", "orden", 2, "valor", Pokedex[ataque]),
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "Defensa", "orden", 3, "valor", Pokedex[defensa]),
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "Ataque esp.", "orden", 4, "valor", Pokedex[ataque_especial]),
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "Defensa esp.", "orden", 5, "valor", Pokedex[defensa_especial]),
    SELECTCOLUMNS(Pokedex, "id", Pokedex[id], "estadistica", "Velocidad", "orden", 6, "valor", Pokedex[velocidad])
)
```

La columna `estadistica` se ordena por `orden`.

## Columnas calculadas en Pokedex

### Total
Suma de las seis estadísticas base.

```
Total =
Pokedex[hp] + Pokedex[ataque] + Pokedex[defensa]
    + Pokedex[ataque_especial] + Pokedex[defensa_especial] + Pokedex[velocidad]
```

### Clase
Sirve como filtro de legendarios.

```
Clase =
SWITCH(
    TRUE(),
    Pokedex[es_mitico], "Mítico",
    Pokedex[es_legendario], "Legendario",
    Pokedex[es_bebe], "Bebé",
    "Común"
)
```

### Tipos
Tipos en español, por ejemplo "Fuego / Volador".

```
Tipos =
VAR t1 = LOOKUPVALUE(DimTipos[tipo_es], DimTipos[tipo], Pokedex[tipo_1])
VAR t2 = LOOKUPVALUE(DimTipos[tipo_es], DimTipos[tipo], Pokedex[tipo_2])
RETURN
    IF(ISBLANK(t2), t1, t1 & " / " & t2)
```

## Medidas de la ficha
Muestran valores solo cuando hay un único Pokémon filtrado; si no, quedan vacías o muestran un mensaje.

```
Pokémon seleccionado =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[nombre]), "Selecciona un Pokémon")
```

```
Tipos del seleccionado =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[Tipos]))
```

```
Altura (m) =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[altura_m]))
```

```
Peso (kg) =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[peso_kg]))
```

```
Total del seleccionado =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[Total]))
```

```
Clase del seleccionado =
IF(HASONEVALUE(Pokedex[id]), SELECTEDVALUE(Pokedex[Clase]))
```

```
Generación del seleccionado =
IF(
    HASONEVALUE(Pokedex[id]),
    LOOKUPVALUE(DimGeneracion[etiqueta], DimGeneracion[generacion], SELECTEDVALUE(Pokedex[generacion]))
)
```

## Medidas de apoyo para los filtros de visual

`Hay selección` se usa como filtro de visual (es 1) en la imagen: la deja en blanco hasta que hay un único Pokémon.

```
Hay selección =
IF(COUNTROWS(ALLSELECTED(Pokedex)) = 1, 1, 0)
```

`Evolución visible` se usa como filtro de visual (mayor que 0) en la tabla de evoluciones. Depende de `Linea`, por eso oculta las filas de otras cadenas.

```
Evolución visible =
IF(HASONEVALUE(Pokedex[id]), COUNTROWS(Linea))
```
