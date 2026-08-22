# From Yesterday — Cheatsheet de modding (Victoria 3 v1.13)

Referencia rápida para el desarrollo del mod. Pensada para tenerla abierta
al lado mientras escribes eventos, no para leerla de un tirón.

**Convención sobre fiabilidad**: donde algo está marcado ✅ lo confirmé
contra `Module:Script_docs` de la wiki (verificado 1.13) o probándolo hoy.
Donde pone ⚠️ es un nombre plausible por convención pero sin verificar —
compruébalo con `script_docs` en consola antes de confiar en él a ciegas.
Cada sección enlaza a la página de la wiki correspondiente para profundizar.

**Índice de páginas raíz de la wiki de modding** (todas cuelgan de aquí,
merece la pena tenerla como favorito):
https://vic3.paradoxwikis.com/Modding

---

## 1. Filosofía del mod

`from_yesterday` gira en torno a una idea central: **las derrotas y malas
decisiones militares tienen que pasar factura política interna**, al
estilo Caporetto/Anual — no solo "pierdes territorio", sino que erosionas
legitimidad, alimentas movimientos radicales, fracturas la relación
gobierno-ejército, etc. Casi todos los eventos van a seguir el mismo
patrón:

```
Hecho militar malo (batalla perdida, guerra perdida, motín, etc.)
        ↓
on_action detecta el hecho
        ↓
Trigger comprueba condiciones políticas (rival, tecnología, gobierno...)
        ↓
Evento presenta opciones = distintas formas de gestionar la crisis
        ↓
Cada opción cambia legitimidad / IGs / movimientos políticos / moral
```

Vale la pena tener esto en la cabeza porque significa que casi todo el
mod se apoya en un mismo puñado de sistemas de Vic3: **legitimidad**,
**interest groups (aprobación e influencia)**, **movimientos políticos**,
**war support**, y **personajes (generales)**.

---

## 2. Estructura de carpetas y convención de nombres

Referencia: https://vic3.paradoxwikis.com/Mod_structure

```
from_yesterday/
├── .metadata/metadata.json
├── common/
│   ├── on_actions/         → fy_*_on_actions.txt
│   ├── modifiers/          → fy_*_modifiers.txt
│   ├── scripted_triggers/  → fy_*_triggers.txt   (triggers reutilizables)
│   ├── scripted_effects/   → fy_*_effects.txt    (effects reutilizables)
├── events/                 → fy_*_events.txt
└── localization/spanish/   → fy_*_l_spanish.yml
```

- **Namespace de eventos**: `fy` a secas es demasiado corto y puede chocar
  con otro mod. Usa algo como `namespace = fy_disasters` o
  `fy_political_fallout` según el bloque temático (uno por "familia" de
  eventos: derrotas en batalla, guerras perdidas, motines, etc.).
- **IDs**: `fy_disasters.1`, `fy_disasters.2`... nunca reutilices un
  número aunque borres el evento — la gente puede tener partidas guardadas
  con eventos pendientes de ese ID.
- **Variables**: prefijo `fy_` siempre, para no chocar con vanilla ni con
  otros mods: `var:fy_own_loss_pct`, no `var:loss_pct`.
- **Modificadores**: `fy_<tema>_<efecto>`, ej. `fy_general_disgraced`,
  `fy_war_support_collapse`.

---

## 3. Modelo mental de scopes

Referencia: https://vic3.paradoxwikis.com/Scope
Lista completa de event targets (todos los `scope_x` posibles):
https://vic3.paradoxwikis.com/Event_target

Piensa en `scope:` como un puntero tipado. Cada effect/trigger solo es
válido en un tipo de scope concreto (su "input"). Encadenas scopes con
`.` o abriendo un bloque:

```
root                          # scope inicial del evento (normalmente country)
  .ig:ig_armed_forces         # country → interest_group
    .leader                   # interest_group → character
      .interest_group         # character → interest_group (vuelta atrás)
```

Scopes clave para este mod:

| Scope | De dónde sale | Para qué lo usarás |
|---|---|---|
| `country` | root de la mayoría de eventos | legitimidad, prestigio, leyes |
| `interest_group` (`ig:ig_x`) | desde `country` | aprobación/influencia de un sector |
| `character` | `leader` de un IG, `scope:attacker`/`scope:defender` de batallas | destituir generales, matar/desacreditar líderes |
| `political_movement` | `political_movement` desde `character`/`interest_group` | alimentar/debilitar radicalismo |
| `battle` / `battle_side` | on_actions de batalla | leer bando, comandantes, provincia |
| `war` | `diplomatic_play` en varios on_actions de guerra | war support, bandos |

### Prefijos de scope "globales" más comunes (no necesitan contexto previo)

Referencia: https://vic3.paradoxwikis.com/Scope#Special_targets

| Prefijo | A qué escopa | Ejemplo |
|---|---|---|
| `c:` | país por su tag | `c:GBR` |
| `s:` | region de estado | `s:STATE_UTAH` |
| `ig:` | interest group del país en scope | `ig:ig_armed_forces` |
| `cu:` | cultura | `cu:welsh` |
| `rel:` | religión | `rel:catholic` |
| `g:` | bien/goods | `g:iron` |
| `law_type:` | tipo de ley | `law_type:law_protected_speech` |
| `ideology:` / `i:` | ideología | `ideology:ideology_liberal` |
| `bt:` | tipo de edificio | `bt:building_tooling_workshops` |
| `scope:` | referencia a un scope guardado antes | `scope:fy_current_enemy` |
| `var:` | variable guardada antes | `var:fy_own_loss_pct` |

---

## 4. On_actions de interés militar (✅ confirmados contra 1.13)

Referencia: https://vic3.paradoxwikis.com/On_action

| On_action | Root | Scopes extra | Cuándo usarlo |
|---|---|---|---|
| `on_battle_ended` | país (atacante o defensor) | `scope:enemy_country`, `scope:battle`, `scope:attacker`, `scope:defender`, `scope:state` | tracking de bajas (ver más abajo) |
| `on_battle_lost` / `on_battle_won` | país | ⚠️ no documentado explícitamente, asumido igual que `on_battle_ended` | disparar el evento de "derrota decisiva" |
| `on_battle_started` | ⚠️ asumido = país, `scope:enemy_country`, `scope:battle` | igual que ended pero al empezar | guardar snapshot pre-batalla |
| `on_war_end` | Diplomatic Play | `scope:actor` (iniciador), `scope:target` | evento "la guerra ha terminado y salió mal" |
| `on_capitulation` | país que capitula | `scope:diplomatic_play` | crisis política por capitulación total |
| `on_peace_agreement_signed_war_participant` | país | — | reacciones a los términos de paz |
| `on_wargoal_enforced` | país contra quien se impuso | `scope:target`, `scope:diplomatic_play`, `scope:wargoal_impact` | humillación diplomática concreta |
| `on_revolution_start` / `on_revolution_end` | país | `scope:target` (país insurgente) | motines/revoluciones como consecuencia |
| `on_law_enactment_fail` | país | — | reformas militares que fracasan tras el desastre |

### Cómo modear un on_action ya existente (sin pisarlo)

```paradox
# en tu propio archivo de common/on_actions/
on_battle_ended = {
    on_actions = {
        fy_on_battle_ended   # nombre único, no tiene que seguir un patrón concreto
    }
}
fy_on_battle_ended = {
    effect = {
        # tus efectos aquí
    }
}
```
Esto es seguro: Paradox permite **añadir** `events`/`on_actions` a un
on_action existente sin sobreescribirlo. Añadir otros bloques (`trigger`,
`weight_multiplier`...) sí da error. Detalle:
https://vic3.paradoxwikis.com/On_action#Modding_on_actions

### Disparar un on_action manualmente desde un effect

```paradox
trigger_event = {
    on_action = fy_on_battle_ended
    days = 5   # opcional, retrasa el disparo
}
```

---

## 5. Triggers que vas a repetir mucho

Referencia general de triggers: https://vic3.paradoxwikis.com/Trigger

```paradox
# Tecnología — https://vic3.paradoxwikis.com/Technology_modding
has_technology_researched = nationalism

# Interest groups — https://vic3.paradoxwikis.com/Interest_group_modding
ig:ig_armed_forces = {
    is_in_government = yes
    approval > 20
}

# Personaje — https://vic3.paradoxwikis.com/Character_modding
scope:general = {
    is_ruler = no
    has_role = general          # ⚠️ verifica el nombre exacto con script_docs
}

# País
legitimacy < 30                 # ✅ event target confirmado, scope country
army_size_including_conscripts  # ✅ confirmado, funciona en CUALQUIER país
num_rivals >= 1                 # ✅ confirmado

# Rivalidad (⚠️ nombre de trigger sin verificar del todo)
scope:enemy_country = { is_rival_of = root }
```

### Operadores lógicos (los vas a usar en casi todos los triggers)

Referencia: https://vic3.paradoxwikis.com/Trigger#Logical_operators

```paradox
trigger = {
    AND = { ... }   # todo debe cumplirse (es el comportamiento por defecto,
                     # normalmente no hace falta escribirlo explícitamente)
    OR  = { ... }    # al menos una condición
    NOT = { ... }    # ninguna de las condiciones dentro
    NOR = { ... }    # equivalente a NOT + OR: ninguna de la lista
    NAND = { ... }   # no todas a la vez
}
```

---

## 6. Effects para "castigar" políticamente (por tema)

Referencia general de effects: https://vic3.paradoxwikis.com/Effect

**Legitimidad y prestigio del gobierno**
https://vic3.paradoxwikis.com/Country_modding
```paradox
add_legitimacy = -10      # ✅ confirmado, scope country
add_prestige = -10        # habitual en vanilla, mismo patrón
```

**Interest groups (aprobación / influencia)**
https://vic3.paradoxwikis.com/Interest_group_modding
```paradox
ig:ig_armed_forces = {
    add_approval = -15          # ⚠️ nombre probable, confírmalo
    add_modifier = {
        name = fy_general_disgraced
        days = 1825
    }
}
```
La influencia normalmente **no se fuerza directamente** — se hace vía
modificadores definidos en `common/modifiers/` (ver
https://vic3.paradoxwikis.com/Modifier_modding y la lista de tipos de
modificador en https://vic3.paradoxwikis.com/Modifier_types) con claves
como `ig_influence_add`, y se los aplicas al IG con `add_modifier`. Es el
patrón que usamos en `fy_disasters_modifiers.txt`.

**Personajes (generales, líderes)**
https://vic3.paradoxwikis.com/Character_modding
```paradox
scope:general = {
    remove_character_role = general   # destituir del mando
    add_trait_character = trait_disgraced   # si tienes un trait custom
}
```

**Movimientos políticos / radicalismo**
Aquí es donde vas a tener que investigar más porque es el sistema menos
documentado de los que usas. No hay página dedicada "Political movement
modding" todavía en la wiki — busca `political_movement` y `radicalism`
directamente en tu `script_docs` local. Puntos de partida:
- `political_movement` scope, target `most_desired_law`
- Interest groups tienen un modificador de radicalismo propio.

**Leyes** (para tus condiciones de "ejército profesional vs milicias")
https://vic3.paradoxwikis.com/Law_modding

**Edificios** (para penalizar industria militar)
https://vic3.paradoxwikis.com/Building_modding

---

## 7. Variables — sintaxis completa

Referencia: https://vic3.paradoxwikis.com/Variable

```paradox
set_variable = {                 # crea o sobreescribe una variable
    name = fy_x
    value = 5                    # puede ser un número, un scope, o un script value
}

change_variable = {              # suma a una variable ya existente
    name = fy_x
    add = 1
}

set_variable = {                 # también soporta subtract/multiply/divide
    name = fy_x                  # directamente dentro de set_variable
    value = var:fy_a
    subtract = var:fy_b
}

multiply_variable = { name = fy_x value = 2 }
divide_variable   = { name = fy_x value = 2 }
add_to_variable_list = { name = fy_list target = scope:target }

clamp_variable = {                # fuerza la variable a un rango
    name = fy_x
    min = 0
    max = 1
}

round_variable = { name = fy_x }  # redondea al entero más cercano
remove_variable = fy_x            # borra la variable

# Lectura
exists = var:fy_x
var:fy_x >= 0.15
```

- Las variables se guardan **en el objeto donde las creas** (país,
  personaje, etc.), no globalmente — si haces `set_variable` en `scope:x`,
  vive ahí, no en `root`.
- `global_var:nombre` existe para variables realmente globales (no
  atadas a ningún objeto). Poco habitual, pero existe:
  https://vic3.paradoxwikis.com/Event_target (busca `global_var`).
- `local_var:nombre` es una variable **temporal dentro del mismo bloque
  de efecto**, útil para cálculos intermedios que no quieres que
  persistan ni un tick.

---

## 8. Guardar y reutilizar scopes

Referencia: https://vic3.paradoxwikis.com/Scope#Saved_scopes

```paradox
scope:enemy_country = {
    save_scope_as = fy_current_enemy   # persiste en el objeto (país) donde lo guardas
}

# más tarde, en OTRO evento sobre el mismo país:
scope:fy_current_enemy = { ... }

# dentro del MISMO efecto, sin necesidad de persistir:
save_temporary_scope_as = fy_temp
scope:fy_temp = { ... }
```

`save_scope_as` es justo lo que usamos en el tracker de batallas para
poder referenciar al enemigo desde `on_battle_ended` después de haberlo
guardado en `on_battle_started`.

---

## 9. Bloques de control e iteradores

Referencia: https://vic3.paradoxwikis.com/Effect y
https://vic3.paradoxwikis.com/Scope#Iterators

```paradox
if = {
    limit = { ... }
    # efectos si se cumple
}
else_if = {
    limit = { ... }
    # efectos si no se cumplió el if pero sí esto
}
else = {
    # efectos si no se cumplió nada de lo anterior
}

# Iteradores más comunes (every_ = todos, random_ = uno al azar,
# ordered_ = todos pero en un orden con "order_by", any_ = trigger)
every_owned_state = { limit = { ... } ... }
every_building = { limit = { building_type = building_arms_industry } ... }
random_country_in_region = { region = ... ... }
any_scope_country = { ... }   # usado como TRIGGER, no effect

hidden_effect = {
    # efectos que NO aparecen en el tooltip del evento (cálculos internos,
    # como nuestro tracking de bajas)
}

custom_tooltip = fy_disasters.1.a.tt   # texto de tooltip manual, para cuando
                                         # el efecto real no debe mostrarse
                                         # literal (p.ej. "fascismo crecerá")
```

---

## 10. Scripted triggers y scripted effects (reutilizar código)

Referencia: https://vic3.paradoxwikis.com/Modding (sección de scripting)
y convención general de Clausewitz — funcionan igual que en otros juegos
Paradox.

```paradox
# common/scripted_triggers/fy_disasters_triggers.txt
fy_is_military_disaster_eligible = {
    has_technology_researched = nationalism
    exists = var:fy_own_loss_pct
    var:fy_own_loss_pct >= 0.10
}

# uso en el evento:
trigger = {
    fy_is_military_disaster_eligible = yes
}
```

```paradox
# common/scripted_effects/fy_disasters_effects.txt
fy_disgrace_general_effect = {
    remove_character_role = general
    add_modifier = { name = fy_general_disgraced days = 1825 }
}

# uso:
scope:general = { fy_disgrace_general_effect = yes }
```

Esto te va a ahorrar muchísima repetición según crezca el mod — en vez de
copiar/pegar el mismo bloque de 10 líneas en cinco eventos distintos, lo
defines una vez y lo llamas por nombre.

---

## 11. Patrón de tracking de batallas (resumen de lo que ya construimos)

Victoria 3 **no expone bajas como número directo** (confirmado contra
`script_docs` 1.13, ver `Module:Script_docs/Event_targets` de la wiki:
https://vic3.paradoxwikis.com/Module:Script_docs/Event_targets).

1. `on_battle_started` → guarda `army_size_including_conscripts` propio
   y del enemigo (`scope:enemy_country`, guardado con `save_scope_as`
   para poder referenciarlo luego).
2. `on_battle_ended` → vuelve a leer esos valores, calcula la diferencia
   = proxy de bajas, guarda `%` propio, `%` enemigo, y la brecha entre
   ambos en variables (con `set_variable`/`clamp_variable`, ver sección 7).
3. El evento de consecuencias políticas dispara leyendo esas variables
   ya calculadas.

Es aproximado (cuenta batallones, no manpower exacto, se puede
contaminar con refuerzos u otros frentes simultáneos) pero es lo más
fiel que permite el motor sin acceso a datos internos de la batalla.

---

## 12. Plantilla de evento base

Referencia: https://vic3.paradoxwikis.com/Event_modding

```paradox
namespace = fy_disasters

country_event = {
    id = fy_disasters.1
    title = fy_disasters.1.t
    desc = fy_disasters.1.d
    picture = "gfx/interface/illustrations/events/generic_war_2.dds"

    trigger = {
        # condiciones políticas + variables calculadas por el on_action
    }

    immediate = {
        # fijar scopes útiles para el resto del evento (scope:general, scope:enemy...)
    }

    option = {
        name = fy_disasters.1.a
        # efectos de la opción A
    }
    option = {
        name = fy_disasters.1.b
        # efectos de la opción B
    }

    after = {
        # limpiar variables temporales usadas solo para este evento
    }
}
```

---

## 13. Flujo de depuración (más fiable que confiar ciegamente en CWTools)

Referencia: https://vic3.paradoxwikis.com/Console_commands y
https://vic3.paradoxwikis.com/Troubleshooting

1. Lanza el juego con `-debug_mode` (opción de lanzamiento en Steam).
2. En consola (tecla `^`), ejecuta `script_docs` → regenera en
   `Documents/Paradox Interactive/Victoria 3/logs/` el listado real y
   actualizado de effects/triggers/scopes válidos para **tu versión
   exacta** del juego. Esto es más fiable que la wiki o que CWTools.
3. Carga tu mod y revisa `logs/error.log` — es la fuente de verdad final,
   el propio parser del juego.
4. `event fy_disasters.1` en consola fuerza que se dispare el evento sin
   esperar a que se den las condiciones — imprescindible para probar
   opciones sin tener que jugar una partida entera hasta que ocurra.
5. Usa CWTools solo como aviso de sintaxis básica (llaves, comas), no
   como validador semántico definitivo — su ruleset para Vic3 va por
   detrás de la versión del juego.

---

## 14. Localización — checklist rápido

Referencia: https://vic3.paradoxwikis.com/Localization y
https://vic3.paradoxwikis.com/Script_localization (para lo dinámico
tipo `$scope:x$` dentro de texto)

- Archivo `.yml` en **UTF-8 con BOM**, no UTF-8 a secas.
- Primera línea siempre `l_spanish:`.
- Una clave por línea: `` clave:0 "texto" ``.
- El `:0` es la versión de la clave — súbelo si cambias el texto y
  quieres forzar refresco de caché de localización.
- Usa `$scope:variable$` para interpolar dentro del texto, ej.
  `"La derrota contra $scope:fy_current_enemy$ ha sido catastrófica."`
- `[scope:x.GetName]` o similar para llamadas a funciones de scripting
  dentro de localización, cuando `$...$` no basta (formato de número,
  etc.) — ver la sección "Customizable Localization" en la página de
  Localization enlazada arriba.

---

## 15. Checklist antes de dar un evento por terminado

- [ ] `namespace` declarado una vez por archivo, coincide con el prefijo de los IDs
- [ ] Todas las claves de loc (`title`, `desc`, cada `option.name`) existen en el `.yml`
- [ ] Variables temporales limpiadas en `after`
- [ ] Trigger probado con `event fy_disasters.1` en consola de debug para forzar su aparición
- [ ] Revisado `error.log` tras cargar la partida con el mod activo
- [ ] Los IDs de leyes/building types/IGs usados están verificados contra `script_docs`, no adivinados