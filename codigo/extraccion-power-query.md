# Extracción y transformación de datos (Power Query)

Todo el proceso de datos del proyecto se hace en Power Query con código M. Los datos vienen de [PokéAPI](https://pokeapi.co/docs/v2), una API pública y gratuita que no pide clave.

**Buenas prácticas con la API**
- La política de uso justo de PokéAPI pide guardar las respuestas en local y no abusar de las peticiones. Por eso los datos se cargan una vez y el modelo hace de caché: no hace falta actualizar el informe.
- Se usa `RelativePath` en `Web.Contents` para que Power BI trate la API como una sola fuente de datos.

**Alcance**
- Pokémon 1 a 386 (generaciones 1 a 3).
- También se incluyen las evoluciones y formas bebé de generaciones posteriores que completan sus líneas evolutivas (por ejemplo, Magneton → Magnezone). Quedan marcadas con `dentro_del_alcance = falso` y no entran en la lista ni en los rankings.

## Organización de las consultas

| Consulta | Tipo | ¿Se carga al modelo? |
|----------|------|----------------------|
| `fGetPokemon`, `fGetSpecies`, `fGetCadena`, `fGetTipo` | Funciones | No |
| `ListaTipos`, `IdsExtra` | Listas | No |
| `Pokemon`, `Especies`, `Cadenas`, `PokemonExtra`, `EspeciesExtra` | Tablas de apoyo | No |
| `Pokedex`, `PokemonTipos`, `EfectividadTipos`, `DimTipos` | Tablas del modelo | Sí |

## Funciones

### fGetPokemon
Estadísticas base, tipos, altura, peso e imagen de un Pokémon.

```
(id as number) as table =>
let
    Source = Json.Document(
        Web.Contents("https://pokeapi.co/api/v2/", [RelativePath = "pokemon/" & Text.From(id)])
    ),
    Stats = Record.FromList(
        List.Transform(Source[stats], each [base_stat]),
        List.Transform(Source[stats], each [stat][name])
    ),
    Tipos = List.Transform(Source[types], each [#"type"][name]),
    Resultado = #table(
        {"id", "nombre", "hp", "ataque", "defensa", "ataque_especial", "defensa_especial",
         "velocidad", "tipo_1", "tipo_2", "altura", "peso", "imagen_url"},
        {{
            Source[id], Source[name],
            Stats[hp], Stats[attack], Stats[defense],
            Stats[#"special-attack"], Stats[#"special-defense"], Stats[speed],
            Tipos{0},
            if List.Count(Tipos) > 1 then Tipos{1} else null,
            Source[height], Source[weight],
            Source[sprites][other][#"official-artwork"][front_default]
        }}
    )
in
    Resultado
```

### fGetSpecies
Generación, clase (legendario, mítico, bebé), nombre en español y cadena evolutiva.

```
(id as number) as table =>
let
    Source = Json.Document(
        Web.Contents("https://pokeapi.co/api/v2/", [RelativePath = "pokemon-species/" & Text.From(id)])
    ),
    IdDesdeUrl = (url as text) as number =>
        Number.FromText(List.Last(List.Select(Text.Split(url, "/"), each _ <> ""))),
    Generaciones = {"generation-i", "generation-ii", "generation-iii", "generation-iv", "generation-v",
                    "generation-vi", "generation-vii", "generation-viii", "generation-ix"},
    NombresEs = List.Select(Source[names], each [language][name] = "es"),
    NombreEs = if List.Count(NombresEs) > 0 then NombresEs{0}[name] else Source[name],
    Padre = Source[evolves_from_species],
    Resultado = #table(
        {"id", "nombre_es", "generacion", "es_legendario", "es_mitico", "es_bebe",
         "cadena_evolutiva_id", "evoluciona_de_id"},
        {{
            Source[id],
            NombreEs,
            List.PositionOf(Generaciones, Source[generation][name]) + 1,
            Source[is_legendary],
            Source[is_mythical],
            Source[is_baby],
            IdDesdeUrl(Source[evolution_chain][url]),
            if Padre = null then null else IdDesdeUrl(Padre[url])
        }}
    )
in
    Resultado
```

### fGetCadena
Aplana una cadena de evolución (estructura anidada) con una función recursiva. Devuelve una fila por Pokémon con su etapa.

```
(idCadena as number) as table =>
let
    Source = Json.Document(
        Web.Contents("https://pokeapi.co/api/v2/", [RelativePath = "evolution-chain/" & Text.From(idCadena)])
    ),
    IdDesdeUrl = (url as text) as number =>
        Number.FromText(List.Last(List.Select(Text.Split(url, "/"), each _ <> ""))),
    Aplanar = (nodo as record, idPadre as nullable number, nivel as number) as list =>
        let
            miId = IdDesdeUrl(nodo[species][url]),
            fila = {[cadena_id = idCadena, especie_id = miId, evoluciona_de_id = idPadre, etapa = nivel]},
            hijos = List.Combine(List.Transform(nodo[evolves_to], each @Aplanar(_, miId, nivel + 1)))
        in
            fila & hijos,
    Filas = Aplanar(Source[chain], null, 1)
in
    Table.FromRecords(Filas)
```

> Aprendizaje: un parámetro de función no debe llamarse igual que un campo del registro que se construye dentro de la misma función (error de nombres en ámbitos anidados).

### fGetTipo
Multiplicador de daño de un tipo atacante contra cada uno de los 18 tipos defensores.

```
(tipoAtacante as text) as table =>
let
    Source = Json.Document(
        Web.Contents("https://pokeapi.co/api/v2/", [RelativePath = "type/" & tipoAtacante])
    ),
    Rel = Source[damage_relations],
    Nombres = (lista as list) as list => List.Transform(lista, each [name]),
    Dobles = Nombres(Rel[double_damage_to]),
    Mitades = Nombres(Rel[half_damage_to]),
    Nulos = Nombres(Rel[no_damage_to]),
    Filas = List.Transform(
        ListaTipos,
        (d) => [
            atacante = tipoAtacante,
            defensor = d,
            multiplicador =
                if List.Contains(Dobles, d) then 2
                else if List.Contains(Mitades, d) then 0.5
                else if List.Contains(Nulos, d) then 0
                else 1
        ]
    )
in
    Table.FromRecords(Filas)
```

## Consultas de apoyo (no se cargan al modelo)

### ListaTipos

```
let
    Tipos = {"normal", "fighting", "flying", "poison", "ground", "rock", "bug", "ghost", "steel",
             "fire", "water", "grass", "electric", "psychic", "ice", "dragon", "dark", "fairy"}
in
    Tipos
```

### Pokemon

```
let
    Ids = {1..386},
    Tabla = Table.Combine(List.Transform(Ids, each fGetPokemon(_)))
in
    Tabla
```

### Especies

```
let
    Ids = {1..386},
    Tabla = Table.Combine(List.Transform(Ids, each fGetSpecies(_)))
in
    Tabla
```

### Cadenas

```
let
    IdsCadenas = List.Distinct(Especies[cadena_evolutiva_id]),
    Tabla = Table.Combine(List.Transform(IdsCadenas, each fGetCadena(_)))
in
    Tabla
```

### IdsExtra
IDs de Pokémon de otras generaciones que aparecen en las líneas evolutivas.

```
let
    Extra = List.Sort(List.Select(List.Distinct(Cadenas[especie_id]), each _ > 386))
in
    Extra
```

### PokemonExtra y EspeciesExtra

```
// PokemonExtra
let
    Tabla = Table.Combine(List.Transform(IdsExtra, each fGetPokemon(_)))
in
    Tabla
```

```
// EspeciesExtra
let
    Tabla = Table.Combine(List.Transform(IdsExtra, each fGetSpecies(_)))
in
    Tabla
```

## Tablas del modelo (se cargan)

### Pokedex
Una fila por Pokémon: estadísticas, tipos, generación, clase, etapa evolutiva, altura en metros, peso en kilogramos e imagen.

```
let
    Base = Table.Combine({Pokemon, PokemonExtra}),
    Esp = Table.Combine({Especies, EspeciesExtra}),
    ConEspecie = Table.ExpandTableColumn(
        Table.NestedJoin(Base, {"id"}, Esp, {"id"}, "esp", JoinKind.Inner),
        "esp",
        {"nombre_es", "generacion", "es_legendario", "es_mitico", "es_bebe", "evoluciona_de_id"}
    ),
    ConCadena = Table.ExpandTableColumn(
        Table.NestedJoin(ConEspecie, {"id"}, Cadenas, {"especie_id"}, "cad", JoinKind.LeftOuter),
        "cad",
        {"cadena_id", "etapa"}
    ),
    ConAlcance = Table.AddColumn(ConCadena, "dentro_del_alcance", each [id] <= 386, type logical),

    // Limpieza
    Ordenado = Table.Sort(ConAlcance, {{"id", Order.Ascending}}),
    SinNombreTecnico = Table.RemoveColumns(Ordenado, {"nombre"}),
    Renombrado = Table.RenameColumns(SinNombreTecnico, {{"nombre_es", "nombre"}}),
    ConAltura = Table.AddColumn(Renombrado, "altura_m", each [altura] / 10, type number),
    ConPeso = Table.AddColumn(ConAltura, "peso_kg", each [peso] / 10, type number),
    SinUnidadesOriginales = Table.RemoveColumns(ConPeso, {"altura", "peso"}),
    Tipado = Table.TransformColumnTypes(
        SinUnidadesOriginales,
        {
            {"id", Int64.Type}, {"nombre", type text},
            {"hp", Int64.Type}, {"ataque", Int64.Type}, {"defensa", Int64.Type},
            {"ataque_especial", Int64.Type}, {"defensa_especial", Int64.Type}, {"velocidad", Int64.Type},
            {"tipo_1", type text}, {"tipo_2", type text}, {"imagen_url", type text},
            {"generacion", Int64.Type},
            {"es_legendario", type logical}, {"es_mitico", type logical}, {"es_bebe", type logical},
            {"evoluciona_de_id", Int64.Type}, {"cadena_id", Int64.Type}, {"etapa", Int64.Type},
            {"dentro_del_alcance", type logical}
        }
    )
in
    Tipado
```

### PokemonTipos
Tabla puente: una fila por combinación Pokémon-tipo (los Pokémon de doble tipo tienen dos filas).

```
let
    Origen = Table.SelectColumns(Pokedex, {"id", "tipo_1", "tipo_2"}),
    Desdinamizada = Table.UnpivotOtherColumns(Origen, {"id"}, "posicion", "tipo"),
    SinNulos = Table.SelectRows(Desdinamizada, each [tipo] <> null),
    Posicion = Table.TransformColumns(
        SinNulos,
        {{"posicion", each if _ = "tipo_1" then 1 else 2, Int64.Type}}
    ),
    Tipado = Table.TransformColumnTypes(Posicion, {{"id", Int64.Type}, {"tipo", type text}})
in
    Tipado
```

### EfectividadTipos
Tabla de 324 filas (18 × 18) con el multiplicador de cada tipo atacante contra cada tipo defensor.

```
let
    Tabla = Table.Combine(List.Transform(ListaTipos, each fGetTipo(_)))
in
    Tabla
```

### DimTipos
Nombre en español y color de cada tipo.

```
let
    Tabla = #table(
        type table [tipo = text, tipo_es = text, color = text],
        {
            {"normal", "Normal", "#A8A77A"}, {"fighting", "Lucha", "#C22E28"},
            {"flying", "Volador", "#A98FF3"}, {"poison", "Veneno", "#A33EA1"},
            {"ground", "Tierra", "#E2BF65"}, {"rock", "Roca", "#B6A136"},
            {"bug", "Bicho", "#A6B91A"}, {"ghost", "Fantasma", "#735797"},
            {"steel", "Acero", "#B7B7CE"}, {"fire", "Fuego", "#EE8130"},
            {"water", "Agua", "#6390F0"}, {"grass", "Planta", "#7AC74C"},
            {"electric", "Eléctrico", "#F7D02C"}, {"psychic", "Psíquico", "#F95587"},
            {"ice", "Hielo", "#96D9D6"}, {"dragon", "Dragón", "#6F35FC"},
            {"dark", "Siniestro", "#705746"}, {"fairy", "Hada", "#D685AD"}
        }
    )
in
    Tabla
```
