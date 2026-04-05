# FORGE — Decisiones de arquitectura

---

## 1. Decisiones tomadas y por qué

---

### HTML single-file como prototipo

**Decisión**: toda la app en un único archivo HTML con JS vanilla.

**Por qué**:
- Velocidad de iteración máxima — un archivo, sin build, sin deploy complejo
- El objetivo es validar con usuarios reales, no construir código escalable todavía
- El entrenador puede usar la app sin instalación, abriendo una URL
- Fácil de compartir, debuggear y modificar en cualquier editor
- Netlify/GitHub Pages lo sirven sin configuración

**Trade-off aceptado**: el archivo tiene >2500 líneas. No escala más allá del piloto. Cuando se valide, se migra a Next.js.

---

### Firebase Realtime Database (no Firestore)

**Decisión**: usar Realtime Database en lugar de Firestore.

**Por qué**:
- Realtime Database funciona con la SDK compat (`firebase-app-compat.js`) directamente en HTML sin build
- Firestore requiere módulos ES6 o configuración más compleja para HTML puro
- Para el volumen del piloto (10-50 atletas) la diferencia de rendimiento es irrelevante
- El plan Spark (gratuito) es suficiente para el piloto completo

**Trade-off aceptado**: Realtime Database convierte arrays en objetos `{0:{...}}`. Se resuelve con `toArr()` + `normalizeMeso()` + `normalizeLog()`. En el SaaS se usará Turso (PostgreSQL) que no tiene este problema.

---

### Leer `forge/logs` completo + filtrar client-side

**Decisión**: el entrenador suscribe a toda la colección `forge/logs` y filtra por `trainerId` en el cliente.

**Por qué**:
- `orderByChild('trainerId')` requiere `.indexOn: ["trainerId"]` en las reglas de Firebase
- Sin ese índice la query devuelve vacío silenciosamente (bug difícil de detectar)
- Para el piloto con <50 atletas el costo de leer todos los logs es mínimo
- Evita complejidad en las reglas de seguridad

**Trade-off aceptado**: ineficiente a escala. En el SaaS, los logs estarán en Turso y se consultarán con SQL indexado.

---

### Código de 6 letras como autenticación del piloto

**Decisión**: no hay login real. El entrenador tiene un código generado una vez, guardado en `localStorage`. El atleta entra con ese código + su nombre.

**Por qué**:
- Firebase Authentication agrega complejidad significativa a un HTML single-file
- Para el piloto entre conocidos, el código es suficiente barrera de acceso
- Cada entrenador tiene datos completamente aislados bajo su código
- El atleta sin el código no puede acceder a ningún dato

**Trade-off aceptado**: no es seguridad de producción. Cualquiera con el código puede escribir datos. Para el SaaS se implementará Firebase Auth o NextAuth.

---

### Modo demo en memoria (sin Firebase)

**Decisión**: el modo demo no toca Firebase — todo en variables JS.

**Por qué**:
- Evita contaminar la base de datos con datos de demo
- Cualquier persona puede ver el demo sin tener cuenta
- `IS_DEMO = true` bloquea todas las escrituras a Firebase
- Los datos demo son suficientemente ricos para mostrar el valor del producto

**Trade-off aceptado**: el demo no persiste entre sesiones ni entre dispositivos.

---

### Semáforo basado en RIR (no en RPE ni en velocidad)

**Decisión**: la lógica del semáforo usa Reps alcanzadas + RIR promedio de las series.

**Por qué**:
- RIR (Reps In Reserve) es el criterio que el entrenador del fundador ya usa
- Es fácil de registrar subjetivamente durante el entrenamiento
- La fórmula es simple y comprensible para el entrenador
- Compatible con el modelo de hipertrofia más común en el mercado argentino

**Trade-off aceptado**: RPE o velocidad de ejecución serían más precisos pero requieren hardware o más complejidad. El feedback del piloto determinará si hay que ajustar la lógica.

---

### 1RM estimado con fórmula Epley

**Decisión**: `1RM = kg * (1 + reps/30)`

**Por qué**:
- Fórmula estándar, ampliamente usada
- Permite comparar progreso entre sesiones con distintas rep ranges
- Cálculo inmediato, sin datos adicionales

**Trade-off aceptado**: es una estimación. Brzycki o Lombardi dan resultados ligeramente distintos. Si el entrenador prefiere otra fórmula se puede cambiar en una línea.

---

### Links YouTube para videos de ejercicios (no video propio)

**Decisión**: el entrenador agrega un link de YouTube al ejercicio. No se hostean videos propios.

**Por qué**:
- Hostear videos requiere almacenamiento, CDN, encoding — costo y complejidad
- YouTube es gratuito, global y ya tiene todos los videos necesarios
- El entrenador puede linkear sus propios videos (muchos tienen canal)
- El atleta ve el video con un tap, sin salir de la app

**Trade-off aceptado**: dependencia de YouTube. Si el video se borra, el link queda roto.

---

### Importador XLSX con SheetJS

**Decisión**: usar SheetJS para leer archivos Excel directamente en el browser.

**Por qué**:
- El entrenador ya tiene sus rutinas en Excel/Google Sheets
- CSV tiene problemas con caracteres especiales en español (tildes, ñ, comillas)
- XLSX es el formato nativo de trabajo del entrenador — cero fricción en la migración
- SheetJS funciona 100% client-side, sin backend

**Formato de columnas esperado**:
```
Semana | Sesion | Ejercicio | Grupo | Series | Reps | Kg | RIR | Tempo | Descanso | Indicaciones
```

---

## 2. Alternativas descartadas

---

### React / Vue / Next.js para el prototipo

**Descartado porque**: requiere build, node_modules, toolchain. Para un piloto de validación agrega 2-3 días de setup sin valor diferencial. Se usará en el SaaS.

---

### Supabase en lugar de Firebase

**Descartado porque**: Supabase requiere server-side o al menos configuración de API keys más compleja para HTML puro. Firebase tiene SDK compat que funciona directo en `<script>` sin build.

---

### localStorage como base de datos principal

**Descartado para el multi-usuario porque**: localStorage es por dispositivo. El entrenador en su compu y el atleta en su celular no pueden compartir datos. Sigue usándose para identidad (`forge_identity`) y sesión activa (`forge_cw_*`).

---

### Google Forms para el feedback

**Descartado porque**: no permite scoring visual de features con pips, no tiene la estética de FORGE, y no da control sobre el formato de los resultados. Se construyó `forge-feedback.html` a medida.

---

### App nativa (React Native / Flutter)

**Descartado para el piloto porque**: requiere publicación en App Store / Play Store, proceso de review, cuentas de developer ($99/año Apple). La PWA cubre el 95% del caso de uso sin ese overhead.

---

### Netlify para el piloto final

**Descartado (plan gratuito agotado) porque**: límite de deploys/usuarios del plan free. Se migró a GitHub Pages que es ilimitado para archivos estáticos.

---

## 3. Zonas protegidas — no tocar sin consulta

---

### `toArr()`, `normalizeMeso()`, `normalizeLog()`

**Por qué protegidas**: son las funciones que resuelven el problema de arrays→objetos de Firebase. Si se modifican o eliminan, todos los datos que vengan de Firebase se rompen silenciosamente.

**Ubicación**: al inicio del bloque de HELPERS en el JS.

---

### Estructura de IDs de Firebase

```
forge/trainers/{TRAINER_CODE}/...
forge/logs/{logId}
```

**Por qué protegida**: cambiar el path rompe todos los datos existentes. El entrenador del piloto ya tiene datos en esta estructura. Cualquier migración requiere script de transformación.

---

### `IS_DEMO` y las guards en `saveMesocycles()` / `saveLog()`

```javascript
function saveMesocycles() {
  if (IS_DEMO) return;  // ← NO TOCAR
  ...
}
```

**Por qué protegida**: si se saca este guard, el modo demo empieza a escribir datos reales en Firebase, contaminando la base de producción.

---

### IDs de tabs en JS (`'session'`, `'plan'`, `'dashboard'`, etc.)

**Por qué protegidos**: son strings usados como IDs de elementos HTML (`section-session`, `tab-session`) y como valores en `activeTab`. Si se les pone tilde o ñ, el routing de la app se rompe.

**Regla**: los IDs internos son siempre ASCII. El texto visible en la UI sí lleva tildes.

---

### GTM ID y GA4 ID

```javascript
GTM-NCX33DBC
G-Z0M5BV38HR
```

**Por qué protegidos**: son los IDs de analytics reales del proyecto. Cambiarlos corta el tracking sin advertencia visible.

---

### Lógica del semáforo en `getExSemaphore()`

**Por qué protegida**: es el diferencial del producto. Cambios en la lógica afectan directamente lo que el entrenador ve y las decisiones que toma. Cualquier modificación debe consultarse con el entrenador primero.

---

## 4. Dependencias críticas entre módulos

---

### `loadTrainerApp()` → `normalizeMeso()` → `renderDashboard()` / `renderPlan()`

```
Firebase retorna mesociclos
  → normalizeMeso() convierte arrays
  → S.mesocycles[] disponible
  → renderDashboard() puede mostrar semáforo
  → renderPlan() puede mostrar ejercicios
```
Si `normalizeMeso()` falla, `renderDashboard()` y `renderPlan()` muestran vacío sin error.

---

### `startSession()` → `LOGS[logId] = log` → `renderWorkout()`

```
startSession() crea el log
  → LOGS[logId] = log  (inmediato, local)
  → saveLog() lo envía a Firebase (asíncrono)
  → renderWorkout() lee LOGS[CW.logId]
```
⚠️ El paso `LOGS[logId] = log` es crítico. Sin él, `renderWorkout()` llega antes que Firebase y muestra spinner infinito.

---

### Firebase Rules → Trainer subscription → `renderDashboard()`

```
forge/logs necesita .read: true (colección completa)
  → logsRef.on('value') recibe todos los logs
  → filtra por trainerId en cliente
  → LOGS[] se puebla
  → renderDashboard() puede calcular semáforo
```
Si las reglas solo permiten leer `$logId` (por ID), el entrenador ve "sin sesiones" aunque existan.

---

### `ASSIGNED_MESO` → `renderSession()` / `renderWorkout()`

```
loadAthleteApp()
  → suscripción a athletes/{ATHLETE_ID}
  → si assignedMesoId existe → fetch mesociclo
  → normalizeMeso() → ASSIGNED_MESO
  → renderSession() puede mostrar la sesión
  → renderWorkout() puede mostrar guía y observaciones
```
Si `ASSIGNED_MESO` es null, el atleta ve "tu entrenador no te asignó un mesociclo".

---

### `CW` (currentWorkout) → localStorage → rehydration

```
startSession() → CW = { logId, exerciseIndex, healthDone }
  → saveCW() → localStorage.setItem('forge_cw_' + ATHLETE_ID)

loadAthleteApp() (al reabrir la app)
  → lee localStorage → restaura CW
  → renderWorkout() continúa desde donde estaba
```
Si se cambia el formato de CW sin migrar localStorage, el atleta pierde la sesión activa al recargar.

---

### `getExSemaphore()` → `lastLog.exercises` → `normalizeMeso().sessions`

```
Para mostrar el semáforo necesita:
  1. lastLog con exercises como ARRAY (normalizeLog)
  2. sess.exercises como ARRAY (normalizeMeso)
  3. exDef encontrado por exerciseId
```
Si cualquiera de los dos es objeto en vez de array, `.filter()` falla y el semáforo muestra vacío.

---

### `downloadXLSXTemplate()` → formato esperado por `rowsFromSheet()`

El orden de columnas de la plantilla descargable DEBE coincidir con los ALIASES:
```
Semana | Sesion | Ejercicio | Grupo | Series | Reps | Kg | RIR | Tempo | Descanso | Indicaciones
```
Si se cambia el orden en la plantilla sin actualizar ALIASES (o viceversa), el importador asigna campos incorrectos silenciosamente.

---

## 5. Fórmulas matemáticas utilizadas

### 1RM Estimado — Fórmula Epley
```
1RM = kg × (1 + reps / 30)
```
**Implementada en**: función `calc1RM(kg, reps)` en `forge-firebase.html`.

**Por qué Epley**: es la fórmula más citada en literatura de entrenamiento de fuerza,
da resultados razonables para rangos de 1-12 reps, y es un cálculo de una línea.
Brzycki (`kg / (1.0278 - 0.0278 × reps)`) y Lombardi (`kg × reps^0.1`) dan resultados
similares en rangos bajos pero divergen en reps altas. Para el uso principal de FORGE
(8-12 reps), Epley es equivalente.

**Zona protegida**: no cambiar sin consultar. Afecta todos los gráficos históricos.
Si el piloto reporta valores inconsistentes, evaluar Brzycki como alternativa.

---

### Semáforo de autorregulación
```
validSets = sets donde reps > 0
repsOk    = TODOS los sets tienen reps >= guideReps   (.every, no .some)
avgRir    = promedio de rir de los sets válidos

SI repsOk  Y  avgRir >= guideRir  → VERDE   (+2.5kg sugerido)
SI repsOk  Y  avgRir <  guideRir  → AMARILLO (mantener carga)
SI NOT repsOk                     → ROJO    (revisar peso o descanso)
SI sin sets válidos               → GRIS    (sin datos)
```
**Implementada en**: función `getExSemaphore(log, exLog, exDef)`.

**Decisiones de diseño críticas**:
- `.every()` para repsOk — **todos** los sets deben alcanzar el objetivo.
  Si el atleta hizo 10/10/8 con objetivo 10 → ROJO porque el tercer set falló.
- RIR se promedia entre todos los sets válidos (no se toma el peor ni el mejor).
- Umbral `avgRir >= guideRir` (mayor o igual). RIR promedio = RIR objetivo → VERDE.

**Zona protegida**: lógica intocable sin consulta. Es el diferencial del producto.

---

### Sugerencia de progresión (+2.5kg)
Valor fijo estándar en entrenamiento de hipertrofia (microload progression).
En el SaaS podría ser configurable por ejercicio (ej: +5kg sentadilla, +1.25kg elevaciones).
Pendiente de feedback del piloto.

---

## 6. Decisiones de autenticación

### Firebase Auth email/password (no Google OAuth)

**Decisión**: usar email/password de Firebase Auth como único método de login.

**Por qué**:
- `signInWithPopup` (Google OAuth) es bloqueado por browsers modernos en GitHub Pages — el popup nunca llega a abrirse
- `signInWithRedirect` falla por "Tracking Prevention blocked access to storage" — GitHub Pages (`github.io`) y Firebase Auth (`firebaseapp.com`) son dominios distintos y el browser bloquea las cookies cross-domain
- Email/password funciona 100% en cualquier hosting, cualquier browser y cualquier dispositivo
- Para el piloto con usuarios conocidos, email/password tiene fricción aceptable

**Trade-off aceptado**: el usuario debe recordar email y contraseña. En el SaaS con dominio propio, Google OAuth funciona sin problemas.

---

### `forge/users/{uid}` como nodo de perfil

**Decisión**: guardar el perfil del usuario en Firebase bajo `forge/users/{uid}` en lugar de localStorage.

**Por qué**:
- localStorage es por dispositivo/browser — el entrenador no podía ver sus datos desde el celular
- Firebase persiste el perfil asociado al UID de Auth — mismo acceso desde cualquier dispositivo
- `onAuthStateChanged` restaura la sesión automáticamente sin que el usuario haga nada

**Estructura**:
```json
forge/users/{uid} = {
  "role": "trainer" | "athlete" | "autonomous",
  "trainerCode": "ABC123",
  "athleteId": "ABC123_uid8chars",
  "name": "Juan García",
  "email": "juan@email.com",
  "createdAt": "ISO timestamp"
}
```

**Zona protegida**: este nodo es la fuente de verdad de la identidad del usuario. Si se corrompe, el usuario no puede entrar a la app. Nunca sobreescribir sin verificar primero.

**Regla Firebase necesaria**:
```json
"forge/users/$uid": {
  ".read": "auth != null && auth.uid == $uid",
  ".write": "auth != null && auth.uid == $uid"
}
```

---

### Rol `autonomous` — trainer + athlete self-vinculado

**Decisión**: el atleta autónomo es técnicamente un entrenador que se asigna a sí mismo como atleta.

**Implementación**:
- Se genera un `TRAINER_CODE` propio (igual que un entrenador)
- `ATHLETE_ID = TRAINER_CODE + "_self"`
- Se crea `forge/trainers/{code}/athletes/{code}_self` apuntando a sí mismo
- El mesociclo activo se auto-asigna al atleta self en `saveMesocycles()`
- `loadAutonomousApp()` combina las vistas de trainer (Plan, Dashboard) + athlete (Sesión, Stats, Salud, Progreso, Historial)

**Trade-off aceptado**: el autónomo tiene datos duplicados como trainer y como atleta bajo el mismo código. En el SaaS se resuelve con el schema de roles múltiples (ver analysis.md sección 8.1).

---

### Cancelar sesión — hard delete del log

**Decisión**: `cancelSession()` hace `DB.ref('forge/logs/' + CW.logId).remove()` — borrado físico.

**Por qué**: un log incompleto cancelado no tiene valor analítico. Dejarlo en Firebase contaminaría las stats y el semáforo. Se pide confirmación antes de borrar.

**Comportamiento**: el log desaparece de Firebase, `LOGS[logId]` se limpia, `CW = null`, vuelve al picker de sesión. El undo NO está disponible para cancelar sesión (a diferencia de borrar ejercicios).

---

### Soft-delete de sesiones del plan — snapshot en logs

**Decisión**: `delSess()` elimina la sesión del array del mesociclo pero los workout_logs existentes conservan el snapshot de `sessionName` y `exerciseId`.

**Por qué**: borrar una sesión del plan no debe borrar el historial del atleta. Los logs ya registrados siguen en `forge/logs` y aparecen en Historial y Progreso. El semáforo para esa sesión muestra gris (sin `exDef`) en lugar de crashear.

**Consecuencia**: `exDef` puede ser `null` en `getExSemaphore()` cuando el ejercicio fue eliminado del plan. La función ya maneja este caso — muestra gris si `!exDef`.

---

## 7. Nuevas variables y persistencia

### `DISMISSED_TIPS` en localStorage

**Key**: `forge_tips_dismissed`
**Formato**: `{tipId: true, ...}`
**Por qué localStorage**: los tips son preferencia de UX, no datos de negocio. No necesitan sincronizarse entre dispositivos. Si el usuario borra la caché, los tips vuelven a aparecer — comportamiento aceptable.
**Funciones**: `buildTip(id, icon, html)` y `dismissTip(id)`.

---

### `historyOpenLog` — estado global de Historial

**Variable**: `var historyOpenLog = null`
**Comportamiento**: almacena el `logId` de la sesión expandida en la pestaña Historial. Al hacer reload se pierde — el historial vuelve a mostrar todo colapsado. Comportamiento esperado y aceptable.


---

## 8. Decisiones de UX — Plan durante sesión activa

### Bloqueo del Plan cuando hay sesión activa (Opción A)

**Decisión**: `renderPlan()` muestra un card de bloqueo si `CW !== null`. El usuario no puede editar el plan mientras entrena.

**Alternativas descartadas**:
- Warning no bloqueante: confuso — el usuario puede creer que los cambios aplican
- Pausar/reanudar sesión: agrega estado complejo (`CW.paused`), más superficie de bugs

**Por qué**: elimina toda la categoría de bugs Plan↔Sesión de una vez. El snapshot `SESSION_MESO_SNAPSHOT` sigue existiendo como seguridad adicional, pero ya no es necesario en el flujo normal.

---

## 9. Decisiones técnicas — Plan (accordion y debounce)

### `openSessions` — persistencia del estado del acordeón

**Variable global**: `var openSessions = {}`

**Clave**: `bodyId` (`sb-{week}-{sessionIndex}`)

**Llenado**: `toggleSess()` actualiza `openSessions[bodyId] = open` en cada toggle.

**Restauración**: `renderPlan()` después de `el.innerHTML = html` recorre `openSessions` y restaura `.open` en body y chevron.

**Por qué**: `saveMesocycles()` → Firebase → listener → `renderPlan()` reconstruye todo el HTML. Sin `openSessions`, el acordeón siempre vuelve cerrado al editar cualquier campo.

---

### Debounce de 800ms en `saveMesocycles()`

**Variable global**: `var saveMesoTimeout = null`

**Implementación**: `clearTimeout(saveMesoTimeout)` + `setTimeout(fn, 800)`.

**Por qué**: cada keystroke en un input del plan disparaba una escritura a Firebase → listener → re-render. Con debounce, Firebase solo se escribe 800ms después de la última tecla. Reduce re-renders innecesarios y escrituras a Firebase.

**Trade-off**: si el usuario cierra el browser dentro de los 800ms posteriores al último cambio, los datos no se guardan. Aceptable para el piloto.
