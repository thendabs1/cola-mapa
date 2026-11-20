# Mapa de cola

Qué hace la cola, dónde corre cada parte y qué conecta con qué. Copia pública, generada con `python3 docs/mapa.py --publico` a partir del código de un repo privado; sin las llaves ni el nombre de las máquinas.

| **14** | **8** | **10** | **5** | **3** | **27** |
|:---:|:---:|:---:|:---:|:---:|:---:|
| rutas | alias | cuentas | CLIs | máquinas | órdenes |

**Índice**: [Piezas y dónde viven](#piezas-y-dónde-viven) · [Un trabajo](#un-trabajo) · [El tic](#el-tic) · [Elección de ruta](#elección-de-ruta) · [Catálogo](#catálogo) · [Consultas](#consultas) · [Webs por el navegador](#webs-por-el-navegador) · [Agentes en el VPS](#agentes-en-el-vps) · [GitHub Actions](#github-actions) · [Estado compartido](#estado-compartido) · [Código](#código)

## Piezas y dónde viven

La cola reparte peticiones a modelos entre tres máquinas, según la cuota de cada cuenta. El código vive en `thendabs1/cola` y el estado en `thendabs1/cola-datos`, que hace de bus: cada máquina hace pull antes de su tic y push después. Colores: azul, el portátil; naranja, el VPS; violeta, los runners de Actions; verde, GitHub; gris, lo de fuera; amarillo, tú.

```mermaid
flowchart TB
  accTitle: Piezas y dónde viven
  tu(["tú"]):::tu
  subgraph sg_p["Portátil · portatil · donde = local"]
    p_cola["~/.local/share/cola · cola.py<br>nuevo · preguntar · aprobar · ver"]:::portatil
    p_timer["cola.timer<br>cola tic cada 5 min"]:::portatil
    p_clis["CLIs<br>claude · opencode · codex · agy"]:::portatil
    p_ff["Firefox + Violentmonkey<br>userscript cola-web 1.30"]:::portatil
    p_panel["panel GNOME<br>cola estado --json"]:::portatil
    p_skill["agentes: Claude Code y Codex<br>skill consultar-cola"]:::portatil
    p_datos[("~/cola-datos")]:::portatil
    p_cache[("~/.cache/cola")]:::portatil
  end
  subgraph sg_v["VPS · vps · donde = vps · shell"]
    v_cola[("~/cola")]:::vps
    v_timer["cola.timer<br>cola tic cada 5 min"]:::vps
    v_clis["CLIs<br>claude · codex · opencode · agy"]:::vps
    v_datos[("~/cola-datos")]:::vps
    v_repos[("/home/cola/repos/*.git<br>remoto vps de tus repos")]:::vps
    v_arboles[("~/.cache/cola/arboles<br>clones en cola/#lt;id#gt;")]:::vps
  end
  subgraph sg_g["GitHub"]
    g_cola[("thendabs1/cola<br>código · CI")]:::github
    g_datos[("thendabs1/cola-datos<br>estado · workflow cola")]:::github
    g_bot[("cuenta bot/#lt;repo#gt;<br>workflows de los agentes")]:::github
  end
  subgraph sg_a["Runners de Actions · donde = actions"]
    a_jobs{{"workflow cola<br>plan → ejecutar → aplicar"}}:::actions
  end
  subgraph sg_x["Fuera"]
    x_modelos(["Anthropic · OpenAI · OpenCode · Google<br>por sus CLIs"]):::fuera
    x_webs(["ChatGPT · Gemini · Copilot<br>por el navegador"]):::fuera
    x_cuota(["endpoints de cuota<br>Claude · Codex"]):::fuera
  end
  tu --> p_cola
  tu --> p_skill --> p_cola
  p_panel --> p_cola
  p_timer --> p_cola
  p_cola <--> p_datos
  p_cola --> p_cache
  p_cola --> p_clis --> x_modelos
  p_cola --> p_ff --> x_webs
  p_cola --> x_cuota
  p_datos <-->|"pull · push"| g_datos
  v_datos <-->|"pull · push"| g_datos
  v_timer <--> v_datos
  v_cola --> v_timer
  v_timer --> v_clis --> x_modelos
  p_cola -->|"ssh"| v_repos
  v_repos --> v_arboles
  v_clis --> v_arboles
  v_clis -.->|"agentes con «todas»"| g_bot
  g_cola -->|"vps-actualizar"| v_cola
  g_cola -->|"checkout"| a_jobs
  g_datos -->|"actions/pendientes.txt"| a_jobs
  a_jobs -->|"commit"| g_datos
  style sg_p fill:transparent,stroke:#2f62b3,stroke-width:2px
  style sg_v fill:transparent,stroke:#b8620b,stroke-width:2px
  style sg_g fill:transparent,stroke:#2e7d43,stroke-width:2px
  style sg_a fill:transparent,stroke:#6b3fb3,stroke-width:2px
  style sg_x fill:transparent,stroke:#6b6b74,stroke-width:2px,stroke-dasharray:4 3
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

| Máquina | nombre | donde | CLIs | Además | Qué hace |
|---|---|---|---|---|---|
| **Portátil** | `portatil` | `local` | claude · opencode · codex · agy | llega al VPS por SSH (`vps`); webs en el perfil de Firefox «default» | Donde trabajas: creas y apruebas trabajos, preguntas al momento y usas las webs por Firefox. Su tic corre los trabajos «local» y «cualquiera», y lee la cuota de Claude, Codex y agy y la publica para las demás. |
| **VPS** | `vps` | `vps` | claude · codex · opencode · agy | `shell = true`: agentes con «todas»; `github`: `gh` como la cuenta bot | Siempre encendido (1 CPU, ~1 GB): su tic corre los trabajos «vps» y «cualquiera», también de noche, y los agentes con shell («todas»). |
| **Runners de Actions** | `actions` | `actions` | — | — | Los jobs plan y aplicar del workflow cola: pasos de tipo comando de los trabajos «actions». Sin CLIs de modelos ni cuentas. |

Ejemplos en `maquina.ejemplo.toml`, `maquina.vps.ejemplo.toml`, `maquina.actions.toml`. Cada máquina tiene el suyo en `~/.config/cola/maquina.toml`, fuera de cualquier repo: dice dónde están sus binarios y sus credenciales.

## Un trabajo

`cola nuevo` deja un trabajo en cola-datos; cualquier tic que pueda correrlo lo coge, paso a paso, y cada paso deja su salida y su línea en el registro. `cola preguntar` es el atajo sin trabajo (ver «Consultas»).

```mermaid
sequenceDiagram
  actor tu as Tú
  participant C as cola (portátil)
  participant D as cola-datos (GitHub)
  participant T as tic (una máquina)
  participant M as CLI del modelo
  tu->>C: cola nuevo "…" --modelo alto
  C->>C: valida el trabajo y, si va a una web, que el prompt cabe
  C->>D: trabajos/#lt;id#gt;/ y control.json, commit y push
  Note over C,D: desde una terminal queda aprobado, desde un agente espera a cola aprobar
  tu->>C: cola aprobar #lt;id#gt; (si hace falta)
  loop cada 5 min, como mucho un paso por pasada
    T->>D: pull --rebase (cola.py fusionar junta estado.json y control.json)
    T->>T: siguiente paso y ruta con cuota
    T->>D: reclama el paso: estado.json en_curso y push
    T->>M: prompt
    M-->>T: respuesta, gasto y, si lo hubo, límite
    T->>D: salidas/, estado.json y registro/, commit y push
  end
  tu->>C: cola ver #lt;id#gt; --salidas
```

El estado del trabajo sale de los de sus pasos (`estado_de`). Se mira en este orden: cancelado, inválido, sin aprobar, pausado y, si no, el peor de sus pasos.

```mermaid
stateDiagram-v2
  [*] --> pendiente: cola nuevo desde una terminal
  [*] --> sin_aprobar: cola nuevo desde un agente
  sin_aprobar --> pendiente: cola aprobar
  pendiente --> esperando: sin ruta con cuota o antes de no_antes
  esperando --> pendiente: reinicio, fin del bloqueo, llega la hora o cola usar-reserva
  pendiente --> en_curso: un tic reclama el paso
  en_curso --> pendiente: quedan pasos o hubo un límite
  en_curso --> sin_aprobar: paso con revisar = true
  en_curso --> hecho: era el último paso
  en_curso --> error: fallo, 5 límites seguidos o un tope
  en_curso --> incierto: el proceso o la máquina desaparecieron
  error --> pendiente: cola reintentar
  incierto --> pendiente: cola reintentar
  pendiente --> pausado: cola pausar
  pausado --> pendiente: cola reanudar
  pendiente --> cancelado: cola cancelar
  hecho --> [*]
  cancelado --> [*]
```

<details>
<summary>Los 10 estados, los 6 tipos de paso, las prioridades y dónde corre</summary>

| Estado | Cuándo |
|---|---|
| `cancelado` | `cola cancelar`: no vuelve a correr |
| `invalido` | `trabajo.toml` no pasa la validación |
| `sin_aprobar` | sin aprobar, cambiado después de aprobarlo, o con un paso «revisar» hecho que aún no has visto |
| `pausado` | `cola pausar` |
| `error` | un paso falló o llegó a un tope; `cola reintentar` |
| `incierto` | el proceso murió o su máquina no volvió: pudo gastar cuota; `cola reintentar` |
| `en_curso` | una máquina ejecuta un paso |
| `esperando` | un paso espera cuota, el fin de un bloqueo o su hora (`no_antes`) |
| `pendiente` | le toca al siguiente tic que pueda correrlo |
| `hecho` | todos los pasos hechos |

| Tipo de paso | Qué hace |
|---|---|
| `normal` | un prompt, una respuesta |
| `planificar` | responde un JSON con subconsultas (esquema fijo) en `salidas/<paso>.json` |
| `abanico` | una llamada por subconsulta del plan, una por tic (hasta 8) |
| `sintetizar` | junta las salidas de los pasos que usa: es el paso `contraste` de `--contrastar` |
| `revisar` | revisa otra salida y acaba en APROBADO o CAMBIOS: es el paso `revision` de `--revisar` |
| `comando` | una orden de shell, solo en GitHub Actions (`--comando`) |

| Prioridad | Qué significa |
|---|---|
| `alta` | va primero y corre aunque no se haya podido leer la cuota (es la de `preguntar` y `despertar`) |
| `normal` | la de siempre |
| `relleno` | solo con cuota leída y cerca de un reinicio, para aprovechar lo que sobra de la ventana |

| donde | Quién lo coge |
|---|---|
| `cualquiera` | la primera que lo reclame (push de su `estado.json`) |
| `local` | solo el portátil |
| `vps` | solo el VPS (los de `--herramientas`) |
| `actions` | solo GitHub Actions (los de `--comando`) |

</details>

## El tic

Lo lanza `cola.timer` cada 5 min en el portátil y en el VPS; `cola.service` lo corta a los 330 min. Ejecuta como mucho un paso por pasada y, mientras ese paso corre, el tic de esa máquina no hace nada más.

```mermaid
flowchart TD
  accTitle: El tic
  ini(["cola tic"]) --> lock{"¿tic.lock libre?"}
  lock -->|"no"| f1["sale: hay otro tic en marcha"]:::no
  lock -->|"sí"| sync["sincroniza cola-datos: commit local, pull --rebase y push<br>sin red: solo los trabajos de esta máquina"]
  sync --> rec["reconcilia: un en_curso de esta máquina es huérfano,<br>y uno ajeno si pasa de su tiempo máximo + 20 min<br>→ incierto"]
  rec --> pp["reintenta los push pendientes de ramas cola/#lt;id#gt;"]
  pp --> blo["bloqueos de hace más de 10 min: mira la cuota real:<br>si hay margen, los levanta, y si está llena, los alarga hasta el reinicio"]
  blo --> lista["trabajos pendientes o esperando de esta máquina o de cualquiera,<br>por prioridad y antigüedad"]
  lista --> cuo["refresca y publica las cuotas que sabe leer (cada 10 min)"]
  cuo --> paso{"siguiente paso del trabajo<br>con los que usa ya hechos"}
  paso -->|"no hay"| sig{"¿queda otro trabajo?"}
  paso --> esp{"¿espera vigente, antes de no_antes,<br>tope de pasos (10) o de USD?"}
  esp -->|"sí"| sig
  esp -->|"no"| eleg{"elegir ruta"}
  eleg -->|"ninguna con cuota"| wait["esperas/#lt;máquina#gt;.json:<br>hasta cuándo y por qué"]
  wait --> sig
  eleg -->|"una ruta"| recl{"donde = cualquiera:<br>reclamar con push"}
  recl -->|"otra máquina lo tiene"| f2["sale sin ejecutar"]:::no
  recl -->|"es nuestro"| ej["ejecuta el paso<br>(tiempo máximo: 30 min por defecto)"]:::si
  ej --> com["commit y push, y sale: un paso por tic"]:::si
  sig -->|"sí"| paso
  sig -->|"no"| f3["commit de los avisos y sale"]
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

## Elección de ruta

Un paso pide un alias (`alto`, `bajo`…) o una ruta. `catalogo.elegir` descarta las rutas que no sirven y toma la primera que tenga cuota. Las cifras son las de por defecto; `config.toml` de cola-datos las cambia por cuenta.

```mermaid
flowchart TD
  accTitle: Elección de ruta
  m(["modelo del paso: alias o ruta"]) --> cand["candidatas: la ruta, o las del alias en su orden"]
  cand --> req{"¿cumple lo que pide el paso?<br>esfuerzo · herramientas · donde<br>gratis (alias gratis) · sin pago"}
  req -->|"ninguna"| otra["en esta máquina no corre:<br>el paso es para otra"]:::no
  req -->|"las que sí"| pref["primero, la de una sesión a medias<br>(misma ruta y máquina: se retoma)"]
  pref --> cada["siguiente ruta, con su cuenta"]
  cada --> blq{"¿bloqueada por un límite?"}
  blq -->|"sí"| prox["descartada"]
  blq -->|"no"| tope{"¿llegó al tope diario?<br>(40 ejecuciones por defecto)"}
  tope -->|"sí"| prox
  tope -->|"no"| con{"¿cuota leída?"}
  con -->|"no, y es Claude o Codex"| alta{"¿prioridad alta,<br>o la de sin_lectura?"}
  alta -->|"no"| prox
  alta -->|"sí"| ok
  con -->|"no, otra cuenta"| rel1{"¿relleno?"}
  rel1 -->|"sí"| prox
  rel1 -->|"no"| ok
  con -->|"sí"| r5{"¿ventana de 5 h por encima del 70 %?"}
  r5 -->|"sí"| prox
  r5 -->|"no"| rs{"¿semana por encima del 80 %?"}
  rs -->|"sí"| prox
  rs -->|"no"| rel2{"¿relleno?"}
  rel2 -->|"no"| ok
  rel2 -->|"sí"| cerca{"¿reinicio en menos de 6 h?"}
  cerca -->|"no"| prox
  cerca -->|"sí"| ok(["esta ruta"]):::si
  prox -->|"quedan"| cada
  prox -->|"no quedan"| espera["el paso espera hasta el primer reinicio<br>o fin de bloqueo que se conozca"]:::no
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

**De dónde sale la cuota.** De una lectura propia, que se guarda 10 min y caduca tras cada paso con la cuenta. Si aquí no se puede leer (el VPS con `setup-token`), vale la que otra máquina publicó en `cuotas/<máquina>.json` si tiene menos de 15 min y es posterior al último paso con esa cuenta. Tras un límite la cuenta queda bloqueada hasta la hora que da el CLI (o 1 h si no la da; 10 min si es saturación del servidor y no el límite de la cuenta, que tampoco cuenta para los 5 límites seguidos que dan el paso por error); pasados 10 min, cada tic mira la cuota real: levanta el bloqueo si hay margen o lo alarga hasta el reinicio de la ventana llena. Uno de más de 6 h (la fecha de un límite semanal, que puede ser el de un modelo) no lo toca. `cola preguntar` elige igual, con prioridad alta.

## Catálogo

De `modelos.toml`: 14 rutas (CLI, cuenta y modelo) en 10 cuentas, y 8 alias. Un alias es una lista ordenada de rutas; el número de cada flecha es su puesto. Las webs solo están en `juez`; si no, se piden por su ruta. En verde azulado, las rutas que corren en el portátil y en el VPS; en azul, solo en el portátil.

```mermaid
flowchart LR
  accTitle: Catálogo
  a_alto(["alto"]):::tu
  a_medio(["medio"]):::tu
  a_bajo(["bajo"]):::tu
  a_gratis(["gratis"]):::tu
  a_revisor(["revisor"]):::tu
  a_juez(["juez"]):::tu
  a_verificador(["verificador"]):::tu
  a_verificador_confianza(["verificador_confianza"]):::tu
  r_claude_pro_opus["claude-pro-opus<br>opus"]:::ambas
  r_claude_team_opus["claude-team-opus<br>opus"]:::ambas
  r_claude_pro_haiku["claude-pro-haiku<br>haiku"]:::ambas
  r_claude_team_haiku["claude-team-haiku<br>haiku"]:::ambas
  r_opencode_beam["opencode-beam<br>opencode/beam-free"]:::ambas
  r_opencode_mimo["opencode-mimo<br>opencode/mimo-v2.6-flash-free"]:::ambas
  r_codex_sol["codex-sol<br>gpt-6.1-sol"]:::ambas
  r_agy_flash["agy-flash<br>gemini-3.8-flash-medium"]:::ambas
  r_agy_flash_high["agy-flash-high<br>gemini-3.8-flash-high"]:::ambas
  r_agy_opus["agy-opus<br>claude-opus-4-6-thinking"]:::ambas
  r_chatgpt_web["chatgpt-web<br>web"]:::portatil
  r_gemini_web["gemini-web<br>web"]:::portatil
  r_copilot_web["copilot-web<br>web"]:::portatil
  r_deepseek_web["deepseek-web<br>web"]:::portatil
  c_claude_pro{{"claude-pro<br>claude"}}:::fuera
  c_claude_team{{"claude-team<br>claude"}}:::fuera
  c_opencode{{"opencode<br>opencode"}}:::fuera
  c_codex{{"codex<br>codex"}}:::fuera
  c_antigravity{{"antigravity<br>agy"}}:::fuera
  c_antigravity_claude{{"antigravity-claude<br>agy"}}:::fuera
  c_chatgpt{{"chatgpt<br>web"}}:::fuera
  c_gemini_web{{"gemini-web<br>web"}}:::fuera
  c_copilot_web{{"copilot-web<br>web"}}:::fuera
  c_deepseek_web{{"deepseek-web<br>web"}}:::fuera
  a_alto -->|"1"| r_claude_pro_opus
  a_alto -->|"2"| r_claude_team_opus
  a_alto -->|"3"| r_agy_opus
  a_medio -->|"1"| r_agy_flash_high
  a_medio -->|"2"| r_codex_sol
  a_medio -->|"3"| r_claude_pro_haiku
  a_bajo -->|"1"| r_agy_flash
  a_bajo -->|"2"| r_claude_pro_haiku
  a_bajo -->|"3"| r_claude_team_haiku
  a_bajo -->|"4"| r_opencode_beam
  a_gratis -->|"1"| r_opencode_beam
  a_gratis -->|"2"| r_opencode_mimo
  a_revisor -->|"1"| r_codex_sol
  a_revisor -->|"2"| r_claude_team_opus
  a_revisor -->|"3"| r_claude_pro_opus
  a_juez -->|"1"| r_claude_pro_opus
  a_juez -->|"2"| r_claude_team_opus
  a_juez -->|"3"| r_codex_sol
  a_juez -->|"4"| r_agy_opus
  a_juez -->|"5"| r_gemini_web
  a_juez -->|"6"| r_chatgpt_web
  a_juez -->|"7"| r_copilot_web
  a_verificador -->|"1"| r_claude_pro_opus
  a_verificador -->|"2"| r_claude_team_opus
  a_verificador -->|"3"| r_agy_opus
  a_verificador_confianza -->|"1"| r_codex_sol
  a_verificador_confianza -->|"2"| r_claude_pro_opus
  a_verificador_confianza -->|"3"| r_claude_team_opus
  a_verificador_confianza -->|"4"| r_agy_opus
  r_claude_pro_opus --> c_claude_pro
  r_claude_team_opus --> c_claude_team
  r_claude_pro_haiku --> c_claude_pro
  r_claude_team_haiku --> c_claude_team
  r_opencode_beam --> c_opencode
  r_opencode_mimo --> c_opencode
  r_codex_sol --> c_codex
  r_agy_flash --> c_antigravity
  r_agy_flash_high --> c_antigravity
  r_agy_opus --> c_antigravity_claude
  r_chatgpt_web --> c_chatgpt
  r_gemini_web --> c_gemini_web
  r_copilot_web --> c_copilot_web
  r_deepseek_web --> c_deepseek_web
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

<details>
<summary>Las 14 rutas</summary>

| Ruta | CLI | Cuenta | Modelo | Esfuerzos | Herramientas | Cobro | Dónde | Roles | En los alias |
|---|---|---|---|---|---|---|---|---|---|
| `claude-pro-opus` | `claude` | `claude-pro` | `opus` | low, medium, high, xhigh, max | ninguna, lectura, edicion, todas | suscripcion | local, vps | diseñar, revisar, sintetizar | alto, revisor, juez, verificador, verificador_confianza |
| `claude-team-opus` | `claude` | `claude-team` | `opus` | low, medium, high, xhigh, max | ninguna, lectura, edicion, todas | suscripcion | local, vps | diseñar, revisar, sintetizar | alto, revisor, juez, verificador, verificador_confianza |
| `claude-pro-haiku` | `claude` | `claude-pro` | `haiku` | — | ninguna, lectura, edicion, todas | suscripcion | local, vps | resumir, clasificar | medio, bajo |
| `claude-team-haiku` | `claude` | `claude-team` | `haiku` | — | ninguna, lectura, edicion, todas | suscripcion | local, vps | resumir, clasificar | bajo |
| `opencode-beam` | `opencode` | `opencode` | `opencode/beam-free` | — | ninguna | gratis | local, vps | investigar, resumir | bajo, gratis |
| `opencode-mimo` | `opencode` | `opencode` | `opencode/mimo-v2.6-flash-free` | — | ninguna | gratis | local, vps | investigar, resumir | gratis |
| `codex-sol` | `codex` | `codex` | `gpt-6.1-sol` | low, medium, high, xhigh | ninguna, todas | suscripcion | local, vps | revisar, diseñar, editar | medio, revisor, juez, verificador_confianza |
| `agy-flash` | `agy` | `antigravity` | `gemini-3.8-flash-medium` | — | ninguna, lectura, edicion | suscripcion | local, vps | investigar, resumir, clasificar | bajo |
| `agy-flash-high` | `agy` | `antigravity` | `gemini-3.8-flash-high` | — | ninguna, lectura, edicion | suscripcion | local, vps | investigar, sintetizar | medio |
| `agy-opus` | `agy` | `antigravity-claude` | `claude-opus-4-6-thinking` | — | ninguna, lectura, edicion | suscripcion | local, vps | diseñar, revisar, sintetizar | alto, juez, verificador, verificador_confianza |
| `chatgpt-web` | `web` | `chatgpt` | `web` | — | ninguna | suscripcion | local | revisar, investigar | juez |
| `gemini-web` | `web` | `gemini-web` | `web` | — | ninguna | suscripcion | local | revisar, investigar | juez |
| `copilot-web` | `web` | `copilot-web` | `web` | — | ninguna | suscripcion | local | revisar, investigar | juez |
| `deepseek-web` | `web` | `deepseek-web` | `web` | — | ninguna | suscripcion | local | revisar, investigar | — |

| CLI | Cómo se lanza |
|---|---|
| `claude` | `claude -p --output-format json`, sin MCP (`--strict-mcp-config`) ni órdenes con barra |
| `codex` | `codex exec --json --ignore-user-config`: sandbox read-only y sin shell, o todo con «todas» |
| `opencode` | `opencode run --standalone --format json`, con el prompt como argumento |
| `agy` | `agy --model … --output-format json`, con el prompt como argumento |
| `web` | Firefox `--new-tab` y el userscript `web/cola-web.user.js` en Violentmonkey |

</details>

## Consultas

`cola preguntar` corre ya en esta máquina y espera la respuesta; `cola nuevo` deja un trabajo que corre cuando haya cuota, quizá en otra máquina. Las dos montan el prompt igual, en este orden (cada pieza solo si toca):

```mermaid
flowchart TB
  accTitle: Cómo se monta el prompt
  n1["nota de «todas» (y de la cuenta bot)<br>solo pasos con herramientas = todas"]
  n2["plantillas/consulta.md<br>instrucciones completas o plantilla"]
  n3["AGENTS.md o CLAUDE.md del repo<br>instrucciones completas · hasta 6 000 caracteres"]
  n4["--contexto: índice, commit y ficheros<br>hasta 80 000 caracteres"]
  n5["pasos/_contexto.md<br>la foto que hizo cola nuevo --contexto"]
  n6["plantilla del paso<br>contrastar · revisar · depurar"]
  n7["pasos/#lt;paso#gt;.md<br>la petición"]
  n8["subconsulta del abanico"]
  n9["salidas de los pasos que usa<br>y su diff si editaron"]
  n10["instrucción del JSON (planificar)"]
  n1 --> n2 --> n3 --> n4 --> n5 --> n6 --> n7 --> n8 --> n9 --> n10
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

| instrucciones | Qué va delante |
|---|---|
| `auto` | `completas` si el paso lleva contexto; si no, `ninguna`, para que las consultas ligeras no paguen nada |
| `completas` | `plantillas/consulta.md` y el `AGENTS.md` o `CLAUDE.md` del repo, recortado |
| `plantilla` | solo `plantillas/consulta.md` |
| `ninguna` | nada |
| con herramientas | nada: el CLI ya lee el repo y su `AGENTS.md` |

```mermaid
sequenceDiagram
  actor tu as Tú
  participant Q as cola preguntar
  participant A as modelo 1
  participant B as modelo 2
  participant J as quien contrasta
  tu->>Q: --modelo A --modelo B --contrastar J --contexto …
  Q->>Q: delante: consulta.md, AGENTS.md y la foto del repo actual
  Q->>Q: rutas distintas con cuota y prioridad alta, y que cada prompt cabe
  par a la vez
    Q->>A: prompt
  and al mismo tiempo
    Q->>B: prompt
  end
  A-->>Q: respuesta 1, guardada en ~/.cache/cola/consultas/
  B-->>Q: respuesta 2
  Q->>J: plantilla contrastar, petición original y respuestas numeradas, sin decir de quién
  J-->>Q: coincidencias, discrepancias, errores y respuesta final
  Q-->>tu: el contraste por la salida estándar y dónde quedó cada respuesta por la de error
  Q->>Q: registro/ y bloqueos/, y commit de cola-datos si no hay un tic en marcha
```

```mermaid
flowchart LR
  accTitle: Pasos de --contrastar y --revisar
  subgraph sg_c["cola nuevo --modelo A --modelo B --contrastar J"]
    c1["p1 · A"] --> cc["contraste · J<br>tipo sintetizar<br>plantilla contrastar"]
    c2["p2 · B"] --> cc
  end
  subgraph sg_r["cola nuevo --modelo A --revisar R"]
    r1["p1 · A"] --> rr["revision · R<br>tipo revisar<br>APROBADO o CAMBIOS"]
  end
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

|  | `cola preguntar` | `cola nuevo` |
|---|---|---|
| Cuándo corre | ya, en esta máquina | cuando un tic lo coja y haya cuota |
| Prioridad | alta | la del trabajo |
| Aprobación | no hace falta: lo que manda está acotado a `--contexto` | desde una terminal, aprobado; desde un agente, `cola aprobar` |
| Varios modelos | a la vez, y el contraste al final | pasos `p1`…`pN` y `contraste`, uno por tic |
| Revisión | — | `--revisar MODELO`: paso `revision` |
| Herramientas | nunca | `--herramientas`, en el VPS |
| Respuesta | salida estándar y `~/.cache/cola/consultas/` | `salidas/` (`cola ver <id> --salidas`) |
| Si una web no cabe | falla antes de abrir ninguna | falla al crear el trabajo, no al correr |

Topes del prompt: `--contexto` hasta 80 000 caracteres; una web admite 100 000 en total, y si contrasta o revisa se guardan 20 000 por respuesta que juzga. El de `cola revisar` recorta el diff según la ruta que lo lea: 100 000 a una web, 90 000 a agy u OpenCode (argumento) y 300 000 a Claude o Codex (stdin).

## Webs por el navegador

Rutas `chatgpt-web`, `gemini-web`, `copilot-web` y `deepseek-web`: solo en el portátil, con Firefox abierto. El prompt no viaja en la URL: la pestaña solo lleva el puerto y una clave tras `#`, que el navegador no manda a la web. Las pestañas escriben de una en una y esperan su respuesta a la vez.

```mermaid
sequenceDiagram
  participant C as cola (adaptadores.py)
  participant L as escuchador 127.0.0.1
  participant F as Firefox
  participant U as userscript cola-web 1.30
  participant W as la web (ChatGPT, Gemini, Copilot o DeepSeek)
  C->>C: espera el turno de envío (flock en web-envio.lock)
  C->>L: abre un puerto libre con una clave de un solo uso
  C->>F: firefox -P perfil --new-tab URL#35;cola=puerto:clave
  F->>U: carga la web y el userscript ve el #35;cola
  U->>L: GET /prompt con la clave (una sola vez)
  L-->>U: el prompt (hasta 100 000 caracteres)
  U->>W: lo escribe
  U->>L: POST /respuesta con la fase «enviando», justo antes del clic
  U->>W: pulsa enviar
  U->>L: POST /respuesta con la fase «enviado»
  L-->>C: suelta el turno (como mucho a los 90 s)
  loop mientras la web genera
    U->>L: latido cada 30 s
  end
  W-->>U: respuesta terminada
  U->>L: POST /respuesta con el texto y la sonda
  L-->>C: la respuesta
  Note over C,L: sin noticias en 5 min, se corta. Si no llegó a enviar, se reintenta una vez en otra pestaña.
  Note over U,W: con --atendido, si se atasca pone «⚠ cola: entra aquí» en el título y avisa con notify-send
```

- Si una web cambia su HTML, el userscript devuelve un error con diagnóstico y deja una **sonda** en `~/.cache/cola/sondas/` (qué casa con cada selector, el HTML de la respuesta y del cuadro, y cuándo pasó cada fase). `cola web-prueba` manda una respuesta conocida a cada web y dice si se leyó bien.
- Reintento: solo si la pestaña pidió el prompt pero no avisó «enviando» (justo antes del clic) y quedan al menos 1 min. Después de enviar no se repite: sería pagar dos veces.
- `cola preguntar` espera a una web al menos 60 min; los latidos permiten cortar antes si la pestaña se cae.
- Sin ventana (#36): con `[chatgpt] sin_ventana = "perfil"`, cada consulta abre una ventana en un Firefox headless de ese perfil, que lanza la primera y sigue corriendo; las ventanas no se frenan como las pestañas ocultas y el userscript las cierra al acabar. `--atendido` va al perfil de siempre.
- `cola web-instalar` abre el userscript del árbol desde el que se lanza, para instalarlo o probar una rama en Violentmonkey (`--sin-ventana`: en el perfil sin ventana, cerrando antes su Firefox headless). `web/sin_ventana.py html|prueba RUTA` abre la web en un Firefox sin ventana con una copia de la sesión: saca el HTML o corre el userscript con la prueba de `web-prueba` (no vale para Copilot).
- `chatgpt/chatgpt-cola.user.js` (2.4) queda solo para `chatgpt -y` desde la terminal.

## Agentes en el VPS

Para dejarle una tarea a Claude o a Codex con el repo delante, sin mandar contexto. Solo viaja lo que tiene commit. El VPS no tiene ninguna credencial de tu cuenta de GitHub; si tiene la de la cuenta bot, el agente puede correr sus propios workflows en la copia del repo que hay allí.

```mermaid
sequenceDiagram
  actor tu as Tú
  participant R as tu repo (portátil)
  participant B as /home/cola/repos/#lt;nombre#gt;.git
  participant D as cola-datos
  participant T as tic en vps
  participant A as agente (Claude o Codex)
  participant G as cuenta bot/#lt;nombre#gt;
  tu->>R: cola vps-alta [--github privado o publico], una vez por repo
  R->>B: ssh: git init --bare y remoto «vps»
  R-->>G: con --github, ssh y gh repo create como la cuenta bot
  tu->>R: cola nuevo "…" --herramientas todas
  R->>B: push de HEAD a cola/#lt;id#gt; (solo lo que tiene commit)
  R->>D: trabajo con donde = vps, repo = el bare, base = ese commit
  T->>D: pull y elige el paso
  T->>T: clona en ~/.cache/cola/arboles/#lt;id#gt; con el .git en ~/.cache/cola/git/#lt;id#gt;
  T->>A: nota de «todas» y prompt, CLI completo sin pedir permiso
  opt si vps tiene cuenta bot
    A->>G: push de cola/#lt;id#gt;, workflows con on push cola/**, gh run watch
  end
  A-->>T: termina, quizá con commits propios
  T->>B: commit de lo que quede y push de cola/#lt;id#gt;, el diff a salidas/
  T->>D: estado, salidas y registro, push
  tu->>R: git fetch vps y git log -p HEAD..vps/cola/#lt;id#gt;
```

| --herramientas | Quién | Qué puede hacer |
|---|---|---|
| `lectura` | Claude | Read, Grep, Glob en el clon, sin shell |
| `edicion` | Claude | además edita; commit y push los hace la cola (nunca main, nunca `--force`) |
| `todas` | Claude o Codex, solo con `shell = true` | el CLI completo: shell, tests, red y el git del clon, durante el tiempo del paso (hasta 300 min con `--tiempo-max`) |

Para poner el código al día después de fusionar en `main`, sin cortar un tic a medias. `cola actualizar` lo hace en las dos máquinas, una detrás de otra: si una falla, la otra sigue y avisa del desfase; si cambió la `@version` del userscript, avisa de que hay que lanzar `cola web-instalar`:

```mermaid
flowchart LR
  accTitle: cola actualizar y cola vps-actualizar
  a["cola actualizar: el VPS por ssh<br>y luego la instalación de aquí<br>(cola vps-actualizar: solo el VPS)"]:::portatil --> b{"coge tic.lock<br>(espera hasta 10 min)"}:::vps
  b -->|"hay un tic largo"| x1["sale: prueba luego"]:::no
  b --> g{"¿en main y sin<br>cambios sin commit?"}:::vps
  g -->|"no"| x4["sale sin tocar nada"]:::no
  g --> c["git fetch y merge --ff-only origin/main"]:::vps
  c -->|"no es avance directo"| x2["sale sin tocar nada"]:::no
  c --> d{"¿importa el código nuevo?"}:::vps
  d -->|"no"| x3["vuelve al commit de antes"]:::no
  d -->|"sí"| e["daemon-reload si cambió systemd/"]:::si
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

## GitHub Actions

Para órdenes de shell (`cola nuevo --comando "pytest -q"`), que nunca corren en el portátil ni en el VPS. Un paso por vuelta, en tres jobs; solo `aplicar` escribe, y no ejecuta nada del trabajo. Cada paso corre en un runner nuevo: lo que necesite del anterior va en un solo comando.

```mermaid
flowchart LR
  accTitle: GitHub Actions
  disp["push a main que cambia<br>actions/pendientes.txt"]:::github
  man["workflow_dispatch<br>a mano o relanzado"]:::github
  subgraph sg_w["workflow cola en thendabs1/cola-datos"]
    plan{{"plan · contents: read<br>checkout de cola-datos y del código con COLA_CODIGO<br>cola.py plan-actions: paso, huella, rev y commit"}}:::actions
    ej{{"ejecutar · sin permisos ni secretos<br>clona el repo del trabajo sin credenciales<br>corre el comando, como mucho 55 min"}}:::actions
    ap{{"aplicar · contents y actions: write<br>valida el artefacto y que el paso sigue igual<br>cola.py aplicar-comando y commit"}}:::actions
  end
  disp --> plan
  man --> plan
  plan -->|"plan"| ej
  ej -->|"artefacto: rc y salida.txt"| ap
  plan -->|"plan"| ap
  ap --> datos[("cola-datos<br>salidas/ y estado.json")]:::github
  ap -.->|"quedan pasos: gh workflow run<br>hasta 20 vueltas seguidas"| man
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

Disparadores (leídos del workflow): `push`, `workflow_dispatch`. El repo del trabajo ha de ser una URL `https://` pública y `base` una rama, una etiqueta o un commit de 40 caracteres.

## Estado compartido

Cada fichero de cola-datos tiene un solo tipo de escritor, para que no se pierdan órdenes ni se ejecute un paso dos veces. Los que dos máquinas pueden tocar a la vez los fusiona git con `cola.py fusionar`: en un paso gana el `rev` mayor, dos reclamaciones a la vez son un conflicto (pierde la que publica después) y, entre dos órdenes, gana la prudente.

```mermaid
flowchart LR
  accTitle: Quién escribe qué en cola-datos
  tu(["tú"]):::tu
  ag["un agente<br>cola nuevo"]:::portatil
  maq["cada máquina<br>en su tic"]:::ambas
  wf{{"workflow cola"}}:::actions
  config[("config.toml")]
  ttoml[("trabajos/#lt;id#gt;/<br>trabajo.toml y pasos/")]
  control[("control.json")]
  estado[("estado.json")]
  salidas[("salidas/")]
  porm[("esperas/ · cuotas/ · bloqueos/ · registro/<br>un fichero por máquina")]
  pend[("actions/pendientes.txt")]
  tu -->|"a mano"| config
  tu -->|"cola nuevo"| ttoml
  ag -->|"sin aprobar"| ttoml
  tu -->|"aprobar · pausar · cancelar"| control
  tu -->|"reintentar"| estado
  maq -->|"el dueño del paso"| estado
  maq --> salidas
  maq --> porm
  maq -->|"al hacer commit"| pend
  pend -.->|"dispara"| wf
  wf -->|"aplicar-comando"| estado
  wf --> salidas
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

| En `~/cola-datos` | Lo escribe | Para qué | Al fusionar |
|---|---|---|---|
| `config.toml` | tú, a mano | reservas, tope diario, ventana de relleno y `sin_lectura` por cuenta; tiempo máximo por paso | git, sin más |
| `trabajos/<id>/trabajo.toml` | `cola nuevo` (tú o un agente) o a mano | el trabajo y sus pasos; su huella es lo que apruebas | git, sin más |
| `trabajos/<id>/pasos/` | `cola nuevo` o a mano | el prompt de cada paso; `_contexto.md`, la foto de `--contexto` | git, sin más |
| `trabajos/<id>/control.json` | tus órdenes: aprobar, pausar, cancelar (el tic, nunca) | si el trabajo puede correr | `cola.py fusionar` |
| `trabajos/<id>/estado.json` | la máquina dueña de cada paso, que sube su `rev`; `cola reintentar` | estado, ruta, máquina, intentos, gasto y rama de cada paso | `cola.py fusionar` |
| `trabajos/<id>/salidas/` | la máquina que ejecutó el paso | respuestas; `.diff` de los pasos que editan; `.json` del plan | git, sin más |
| `esperas/<máquina>.json` | cada máquina, la suya | por qué y hasta cuándo espera allí un paso | un fichero por máquina |
| `codigo/<máquina>.json` | el tic de cada máquina, al empezar, si cambió | el commit del código que corre y si tiene cambios sin commit: `cola estado` y `cola seguir` avisan si no coinciden (#13) | un fichero por máquina |
| `cuotas/<máquina>.json` | cada máquina que sabe leer una cuota | la cuota de Claude, Codex y agy para las que no saben | un fichero por máquina |
| `cuotas/historial/<día>-<máquina>.jsonl` | cada máquina que sabe leer una cuota, al cambiar la lectura | la historia de las cuotas de Claude, Codex y agy (#101) | un fichero por máquina |
| `bloqueos/<máquina>.json` | cada máquina, la suya | cuentas que dieron un límite: hasta cuándo, y si se levantó | un fichero por máquina |
| `registro/<día>-<máquina>.jsonl` | cada máquina, la suya | una línea por llamada: ruta, resultado, gasto, duración, cuota antes y después | un fichero por máquina |
| `afirmaciones/<día>-<máquina>.jsonl` | cada máquina, la suya (`cola revisar` de un PR) | una línea por revisión y otra por hallazgo, con lo comprobado (#55) | un fichero por máquina |
| `actions/pendientes.txt` | la cola al hacer commit, si cambia la lista | dispara el workflow cola | `cola.py fusionar` |
| `trabajos/<id>/bench.json` | `bench/bench.py montar` | de qué caso, versión y brazo es un trabajo de la batería (#85) | git, sin más |
| `bench/<día>-<máquina>.jsonl` | `bench/bench.py puntuar` | una línea por vuelta de la batería: cada fase con su gasto acumulado y su acierto contra el oro (#85) | un fichero por máquina |

| En `~/.cache/cola`, en cada máquina | Para qué |
|---|---|
| `tic.lock` | cerrojo: un tic a la vez (`actualizar` y `vps-actualizar` también lo cogen) |
| `web-envio.lock` | turno de envío de las webs: una pestaña escribe a la vez |
| `firefox-sin-ventana.lock` | que dos consultas no lancen a la vez el Firefox sin ventana (#36) |
| `cuotas.json` | caché de las lecturas de cuota (10 min) |
| `publicar-cuotas.lock` | cerrojo: publicar en `cuotas/<máquina>.json` (el tic y `cola cuotas` a la vez) |
| `consultas` | cada consulta de `cola preguntar` entera (prompt, respuesta, la otra lectura de una web y `meta.json`); el tic borra las de más de 30 días salvo las de `cola conservar` |
| `preguntas` | respuestas de `cola preguntar` de antes de `consultas` (el tic las borra a los 30 días) |
| `en-marcha` | un fichero por consulta que corre en esta máquina (ruta, paso, origen, sesión, pid); se borra al quedar en el registro, y uno sin terminar, al día (`estado --json`: `consultas_en_marcha`) |
| `revisiones` | salida de cada `cola revisar` que lanza el tic por un comentario |
| `github.json` | comentarios ya vistos por repo de `[revisiones] vigilar`, y desde cuándo mirar |
| `github-abiertos.json` | PRs (y con la CI en rojo) e issues abiertos por repo de `[revisiones] vigilar`, para la cabecera de `cola seguir` (los cuenta el tic) |
| `sondas` | diagnóstico del userscript cuando una web falla (o con `--depurar`) |
| `crudo` | salida tal cual de cada CLI, por trabajo |
| `arboles` | clones de los repos de los trabajos, en la rama `cola/<id>` |
| `git` | el `.git` de cada clon, fuera del árbol que ve el agente |
| `revisadas.jsonl` | las revisiones hechas, por su huella, para no repetirlas sin `--repetir` (#52, C7); solo de esta máquina |
| `clones` | clon parcial y sin árbol de cada repo revisado (`<dueño>/<repo>.git`), para comprobar las citas con git (#52) |
| `vacio` | directorio vacío donde corren los pasos sin herramientas |
| `tmp` | temporales de los CLIs |

## Código

Python 3.14, solo biblioteca estándar. `cola.py` es el CLI y el planificador; los demás módulos no se importan entre sí.

```mermaid
flowchart LR
  accTitle: Código
  m_cola_py["cola.py"]:::portatil
  m_adaptadores_py["adaptadores.py"]:::portatil
  m_catalogo_py["catalogo.py"]:::portatil
  m_cuotas_py["cuotas.py"]:::portatil
  m_revisiones_py["revisiones.py"]:::portatil
  m_afirmaciones_py["afirmaciones.py"]:::portatil
  m_contexto_py["contexto.py"]:::portatil
  m_red_py["red.py"]:::portatil
  m_lotes_py["lotes.py"]:::portatil
  m_seguir_py["seguir.py"]:::portatil
  m_cola_py --> m_adaptadores_py
  m_cola_py --> m_afirmaciones_py
  m_cola_py --> m_catalogo_py
  m_cola_py --> m_contexto_py
  m_cola_py --> m_cuotas_py
  m_cola_py --> m_lotes_py
  m_cola_py --> m_revisiones_py
  m_cola_py --> m_seguir_py
  m_revisiones_py --> m_afirmaciones_py
  m_revisiones_py --> m_contexto_py
  m_afirmaciones_py --> m_red_py
  m_contexto_py --> m_afirmaciones_py
  m_lotes_py --> m_afirmaciones_py
  modelos[/"modelos.toml"/]
  maquina[/"~/.config/cola/maquina.toml"/]
  plantillas[/"plantillas/*.md"/]
  datos[("~/cola-datos")]
  cache[("~/.cache/cola")]
  clis(["claude · codex · opencode · agy"]):::fuera
  ff(["Firefox + cola-web.user.js"]):::fuera
  apis(["endpoints de cuota"]):::fuera
  m_catalogo_py --> modelos
  m_cola_py --> maquina
  m_cola_py --> plantillas
  m_cola_py --> datos
  m_cola_py --> cache
  m_adaptadores_py --> clis
  m_adaptadores_py --> ff
  m_cuotas_py --> apis
  classDef tu fill:#fff1b8,stroke:#9a7b00,color:#3d3000
  classDef portatil fill:#dbe8fb,stroke:#2f62b3,color:#0f2a55
  classDef vps fill:#fde3c8,stroke:#b8620b,color:#4a2600
  classDef ambas fill:#d5ecee,stroke:#2f7f88,color:#0f3337
  classDef actions fill:#ebdffb,stroke:#6b3fb3,color:#2b1250
  classDef github fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef fuera fill:#ececee,stroke:#6b6b74,color:#222226
  classDef si fill:#d9f0dc,stroke:#2e7d43,color:#123d1d
  classDef no fill:#fbe0de,stroke:#b3423b,color:#4a1210
```

| Orden | Qué hace | Desde dónde | Qué toca |
|---|---|---|---|
| `nuevo` | prompt suelto: trabajo de un paso | portátil o un agente | `trabajos/<id>/` en cola-datos; con `--herramientas`, sube HEAD al remoto `vps` |
| `preguntar` | consulta síncrona: corre ya, sin cola, e imprime la respuesta | portátil o un agente | `registro/`, `bloqueos/` y `~/.cache/cola/consultas/` |
| `revisar` | revisa un issue o PR de GitHub con varios modelos (por defecto, las tres webs; un issue lo contrasta juez y en un PR, verificador verifica cada hallazgo) y, con --publicar, deja el resultado en un comentario | portátil (a mano, o el tic por un comentario «/cola revisar») | como `preguntar`; con `--publicar`, un comentario en el issue o PR con el `gh` del portátil |
| `web-prueba` | manda una respuesta conocida a las webs y mira si se lee bien | portátil | como `preguntar`, con la sonda siempre |
| `web-instalar` | abre web/cola-web.user.js de este árbol en Firefox para instalarlo o actualizarlo en Violentmonkey | portátil, desde cualquier árbol | nada: abre el userscript en Firefox y lo instalas tú |
| `vps-alta` | crea el repo del VPS para el repo actual y le añade el remoto vps | portátil, dentro del repo | por SSH, el repo bare del VPS; el remoto `vps`; con `--github`, el repo de la cuenta bot |
| `actualizar` | pone en origin/main el VPS y luego la instalación de esta máquina (~/.local/bin/cola), sin pararlas a mitad de un tic | portátil (en el VPS, solo él) | por SSH, `~/cola` del VPS, y luego la instalación de aquí, adonde apunta `~/.local/bin/cola` (y `daemon-reload` si cambió `systemd/`) |
| `vps-actualizar` | pone el código del VPS en origin/main, sin pararlo a mitad de un tic | portátil | por SSH, `~/cola` del VPS (y `daemon-reload` si cambió `systemd/`) |
| `despertar` | llamada mínima a una hora para abrir la ventana de 5 h | portátil | un trabajo de prioridad alta con `no_antes` |
| `lista` | lista los trabajos y su estado | portátil o VPS | solo lee |
| `ver` | un trabajo: pasos, estado y, con `--salidas`, las respuestas | portátil o VPS | solo lee |
| `tic` | una pasada: ejecuta un paso, o hasta [general] a_la_vez a la vez | systemd (portátil y VPS) o a mano | `estado.json`, `salidas/`, `esperas/`, `cuotas/`, `bloqueos/`, `registro/`; commit y push; borra las consultas viejas de la caché |
| `estado` | resumen de la cola; con `--json`, para el panel GNOME | panel GNOME o portátil | solo lee |
| `cuotas` | vuelve a leer las cuotas que esta máquina sabe leer (las de lectura más vieja que --edad) y las enseña; con --json, como «cuotas» de estado --json (Super+U del panel) | panel GNOME o portátil | `~/.cache/cola/cuotas.json`, `cuotas/` (sin commit: lo sube el tic); renueva tokens OAuth de Claude |
| `seguir` | en vivo: los trabajos sin acabar y los hechos hace poco, cada uno como la cadena de sus pasos con colores y flechas; con teclas para pausar, aprobar, reintentar, cancelar, lanzar un tic y ver las cuotas; / todas las órdenes y buscar; q o Ctrl-C para salir | portátil o VPS, en una terminal | lee (el cerrojo del tic, en `/proc/locks`); con `--pull` o `u`, `git pull` de cola-datos si no hay tic; sus teclas, lo de `pausar`, `reanudar`, `cancelar`, `aprobar`, `reintentar` y `usar-reserva` (commit solo sin tic) y `t` lanza `cola tic` |
| `pausar` | el trabajo no sigue tras el paso en marcha | portátil | `control.json` |
| `reanudar` | quita la pausa | portátil | `control.json` |
| `cancelar` | el trabajo no vuelve a correr | portátil | `control.json` |
| `aprobar` | aprueba el trabajo tal como está (su huella) y los pasos «revisar» hechos | portátil | `control.json` |
| `reintentar` | vuelve a poner en pendiente los pasos en error o inciertos | portátil | `estado.json` |
| `usar-reserva` | deja que un paso gaste la reserva de cuota (5 h y semanal) una vez, en vez de esperar al reinicio; el límite de verdad y el tope diario siguen mandando | portátil | `control.json` |
| `conservar` | que el tic no borre una consulta de preguntar a los 30 días | portátil o VPS | el fichero `CONSERVAR` de una carpeta de `~/.cache/cola/consultas/` |
| `informe` | llamadas, tiempo, gasto y cuota por ruta | portátil | solo lee `registro/` |
| `catalogo` | las rutas y su cuota; `--actualizar` lista modelos de los CLIs que faltan en `modelos.toml` | portátil | solo lee |
| `plan-actions` | uso interno (workflow): JSON del siguiente paso de Actions | job plan del workflow cola | solo lee |
| `aplicar-comando` | uso interno (workflow): guarda el resultado de un paso | job aplicar del workflow cola | `salidas/`, `estado.json` |
| `fusionar` | uso interno: driver de git para estado.json y control.json | git, al fusionar cola-datos | `estado.json`, `control.json` y `actions/pendientes.txt` |

<details>
<summary>Topes y tiempos (leídos del código)</summary>

| Nombre | Valor | Dónde | Para qué |
|---|---:|---|---|
| `CONTEXTO_MAX` | 80 000 caracteres | `cola.py` | tope de lo que mete `--contexto` en un prompt |
| `WEB_MAX` | 100 000 caracteres | `adaptadores.py` | prompt máximo de una web |
| `PROMPT_MAX_CLI` | 300 000 caracteres | `cola.py` | prompt de una revisión a un CLI que lo lee por stdin: el diff, entero hasta aquí |
| `MARGEN_REVISION` | 20 000 caracteres | `cola.py` | lo que se guarda por respuesta si contrasta o revisa una web |
| `CONVENCIONES_MAX` | 6 000 caracteres | `cola.py` | `AGENTS.md` o `CLAUDE.md` del repo, recortado |
| `SALIDA_MAX` | 20 000 caracteres | `cola.py` | salida de un comando o diff que se guarda |
| `ARG_MAX_PROMPT` | 120 000 bytes | `adaptadores.py` | prompt de OpenCode y agy, que va como argumento |
| `TIEMPO_MAX_MIN` | 30 min | `cola.py` | tiempo máximo por paso si `config.toml` no dice otro |
| `TIEMPO_MAX_TOPE` | 300 min | `cola.py` | tope de `tiempo_max_min` de un trabajo |
| `TIEMPO_MAX_WEB_MIN` | 60 min | `cola.py` | tiempo mínimo de `cola preguntar` a una web |
| `TIEMPO_ACTIONS_MAX` | 55 min | `cola.py` | tope de un comando en Actions |
| `TOPE_PASOS` | 10 | `cola.py` | ejecuciones por trabajo, reintentos incluidos, si no dice otro |
| `MAX_ABANICO` | 8 | `cola.py` | subconsultas de un abanico |
| `INTENTOS_LIMITE` | 5 | `cola.py` | límites seguidos antes de dar el paso por error |
| `ESPERA_LIMITE` | 1 h | `cola.py` | bloqueo tras un límite si el CLI no dice hasta cuándo |
| `ESPERA_TRANSITORIO` | 10 min | `adaptadores.py` | bloqueo si el error es saturación del servidor, no el límite de la cuenta |
| `MARGEN_LEVANTAR` | 10 min | `cola.py` | tras un bloqueo, cuándo empieza a mirarse la cuota real para levantarlo o alargarlo |
| `BLOQUEO_LARGO` | 6 h | `cola.py` | un bloqueo más largo (la fecha de un límite semanal) no lo toca la cuota real |
| `MARGEN_HUERFANO` | 20 min | `cola.py` | margen sobre el tiempo máximo para dar por caído un paso de otra máquina |
| `CUOTA_VALIDA` | 15 min | `cola.py` | antigüedad máxima de una cuota leída en otra máquina |
| `CACHE_TTL` | 10 min | `cuotas.py` | caché de una lectura de cuota |
| `RESERVA_5H` | 30 % | `catalogo.py` | de la ventana de 5 h que la cola no toca, si `config.toml` no dice otro |
| `RESERVA_SEMANA` | 20 % | `catalogo.py` | de la semana que la cola no toca |
| `TOPE_DIARIO` | 40 | `catalogo.py` | ejecuciones al día por cuenta |
| `VENTANA_RELLENO_H` | 6 h | `catalogo.py` | antes de un reinicio, cuándo corre «relleno» |
| `WEB_TURNO_MAX` | 90 s | `adaptadores.py` | turno de envío de una pestaña |
| `WEB_INACTIVIDAD` | 5 min | `adaptadores.py` | sin noticias del userscript: se corta |
| `WEB_REINTENTO_MIN` | 1 min | `adaptadores.py` | lo que ha de quedar para reintentar una web que no llegó a enviar |
| latido | 30 s | `web/cola-web.user.js` | el userscript da señales mientras espera la respuesta |
| `OnUnitActiveSec` | 5 min | `systemd/cola.timer` | cada cuánto hay tic |
| `TimeoutStartSec` | 330 min | `systemd/cola.service` | tope de un tic |
| vueltas | 20 | `actions/cola-datos.yml` | relanzamientos seguidos del workflow cola |

</details>

<details>
<summary>Los 38 ficheros y grupos del repo</summary>

| Fichero | Qué es |
|---|---|
| `cola.py` | CLI y planificador: trabajos, tic, git de cola-datos, clones, preguntar, Actions y fusión |
| `adaptadores.py` | un adaptador por CLI (cómo se lanza y cómo se lee su salida) y las webs: escuchador y turno |
| `catalogo.py` | carga `modelos.toml` y elige la ruta: requisitos, cuota, reservas y topes |
| `cuotas.py` | lee la cuota de Claude y de Codex de sus endpoints, y la de agy con `agy -p /usage`, con caché; renueva el token OAuth de Claude si caduca |
| `revisiones.py` | issues y PRs de GitHub: los lee con `gh`, da forma al comentario, lo publica y entiende «/cola revisar» |
| `afirmaciones.py` | lo común a revisiones y procesos (#52, #56): qué se envió (manifiesto), la afirmación y `comprobar` con sus fuentes (git y web; la web, con caché en `http/` y registro en `http.jsonl` de la caché) |
| `lotes.py` | lotes con esquema y criterios (#56): el sobre, el esquema de los datos (un subconjunto de JSON Schema), leer lo que vuelve y los criterios `responde`, `cita` y `juntos`, con `cumple` en la tabla `afirmaciones/` |
| `seguir.py` | cómo se pintan `cola seguir` y la cadena de `cola ver`: trabajos agrupados por lo que piden, pasos con colores y flechas, tabla de pasos, lotes, cuotas en barras con su ritmo, carriles de llamadas por cuenta, llamadas por día en cuadros verdes, la pestaña Cola (orden del tic, esperas por máquina, commits de cola-datos), la pestaña Consultas (cada llamada del registro, de dónde salió y sobre qué, con filtro) y la pestaña Sesiones (las sesiones vivas de Claude Code y Codex con sus consultas), pestañas, cabecera (plegable), la cinta de órdenes, la línea del elegido y la ventana «/» (todas las órdenes y buscar) |
| `red.py` | bajar una URL que da un modelo con guardas contra SSRF (#56): solo IPs públicas, conexión a la IP ya mirada, topes, en un subproceso que se mata |
| `contexto.py` | contexto gratis de una revisión de PR (#53): funciones que cambian, lo que llaman, quién las llama e historia, con `ast` y git |
| `modelos.toml` | catálogo: rutas (CLI, cuenta, modelo) y alias |
| `maquina.ejemplo.toml` | ejemplo de `~/.config/cola/maquina.toml` del portátil |
| `maquina.vps.ejemplo.toml` | ejemplo de `~/.config/cola/maquina.toml` del VPS |
| `maquina.actions.toml` | la de los runners de Actions (la usa el workflow cola) |
| `datos.gitattributes` | qué ficheros de cola-datos fusiona `cola.py fusionar` |
| `.gh-dash.yml` | gh-dash (`gh dash` dentro del repo): PRs e issues, con R = `cola revisar` y S = `cola seguir` |
| `plantillas/*.md` | plantillas de prompt (ver «Consultas») |
| `plantillas/LEEME.txt` | cómo se usan las plantillas y la licencia de `depurar.md` |
| `web/cola-web.user.js` | userscript de ChatGPT, Gemini, Copilot y DeepSeek para la cola |
| `web/sin_ventana.py` | una web en un Firefox sin ventana con una copia de la sesión: su HTML o el userscript de verdad |
| `chatgpt/chatgpt-cola.user.js` | userscript de ChatGPT para `chatgpt -y` desde la terminal |
| `skill/consultar-cola/SKILL.md` | skill de Claude Code y Codex, el índice: cuándo usar `preguntar`, `nuevo`, `revisar` o Multi-CLI |
| `skill/consultar-cola/*.md` | el detalle de la skill, que se lee cuando toca: webs, contexto, encargos y ejemplos |
| `systemd/cola.service` | una pasada de `cola tic`, con tope de tiempo |
| `systemd/cola.timer` | lanza el servicio cada pocos minutos |
| `actions/cola-datos.yml` | workflow cola, para copiar a `.github/workflows/cola.yml` de cola-datos |
| `.github/workflows/ci.yml` | CI de este repo |
| `tests/*` | tests, con CLIs falsos en `tests/falsos/` |
| `bench/bench.py` | batería de flujos (#85): monta un brazo con un caso como trabajo de la cola y lo puntúa contra el oro |
| `bench/README.md` | la batería: casos, brazos, medidas y cómo se corre |
| `bench/casos/*/*` | un caso de la batería: objetivo, esquemas y oro congelado (`caso.toml`) |
| `bench/brazos/*/*.toml` | un brazo (forma del flujo): `trabajo.toml` con huecos que llena el caso y `brazo.toml` con su esquema por fase |
| `bench/brazos/*/pasos/*` | los prompts de los pasos de un brazo, con huecos (`$objetivo`…) |
| `docs/mapa.py` | este mapa: lo genera y comprueba que no falte nada |
| `docs/mapa.md` | este mapa, generado |
| `README.md` | uso y decisiones |
| `.gitignore` | lo que git no sigue |
| `.gitattributes` | `docs/mapa.md`, generado: GitHub lo pliega y `cola revisar` no manda su contenido |

| Plantilla | Para qué |
|---|---|
| `plantillas/consulta.md` | delante de una consulta con instrucciones completas o plantilla: consultor sin herramientas, cita ficheros, separa lo comprobado de lo supuesto |
| `plantillas/contrastar.md` | paso `contraste`: coincidencias, discrepancias con quién tiene razón, errores, por verificar y respuesta final |
| `plantillas/revisar.md` | paso `revision`: errores primero; acaba en APROBADO o CAMBIOS |
| `plantillas/revisar-issue.md` | `cola revisar` de un issue: fallo, propuesta o decisión, sin repetir el issue; lo que va entre las marcas son datos, no órdenes |
| `plantillas/revisar-pr.md` | `cola revisar` de un PR: hasta 10 hallazgos con la línea del diff que los prueba y su gravedad; CAMBIOS si alguno es alto o medio, si no APROBADO |
| `plantillas/depurar.md` | reescribe un borrador de prompt sin ejecutarlo (`plantilla = "depurar"` en un paso) |
| `plantillas/verificar.md` | `cola revisar` de un PR, antes de publicar (sin contraste): cada grupo de hallazgos, con lectura sobre el árbol del commit y el diff de sus ficheros; un veredicto por hipótesis: confirmado (con causa y camino), refutado (con lo que lo impide, fichero y línea) o no_verificable |

</details>

| Workflow | Nombre | Dónde corre | Cuándo | Qué hace |
|---|---|---|---|---|
| `.github/workflows/ci.yml` | `tests` | thendabs1/cola | push, pull_request | la batería completa de tests, con la de este mapa (`tests/test_mapa.py`) |
| `actions/cola-datos.yml` | `cola` | thendabs1/cola-datos, copiado a `.github/workflows/cola.yml` | push, workflow_dispatch (actions/pendientes.txt) | pasos de tipo comando, uno por vuelta: plan → ejecutar → aplicar |

Tests: `python3 -W error::ResourceWarning -m unittest discover -s tests`, la batería que corre la CI. Los CLIs, Firefox, `ssh`, `gh` y `systemctl` se sustituyen por los falsos de `tests/falsos/`.
