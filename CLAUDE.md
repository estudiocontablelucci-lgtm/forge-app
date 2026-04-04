# FORGE — Training Tracker
## Instrucciones para Claude

---

## ¿Qué es este proyecto?

FORGE es un sistema de seguimiento de entrenamiento para gimnasios y entrenadores personales.
Tiene dos capas:

1. **Prototipo HTML** (`forge-firebase.html`) — app funcional single-file con Firebase Realtime Database
2. **Futuro SaaS** — Next.js + Turso + NextAuth, aún no construido

El objetivo actual es **validar con entrenadores reales** antes de construir el SaaS.

---

## Stack del prototipo

| Capa | Tecnología |
|---|---|
| Frontend | HTML + CSS + JS vanilla (sin frameworks) |
| Base de datos | Firebase Realtime Database |
| Hosting | GitHub Pages / Netlify |
| Analytics | Google Tag Manager (GTM-NCX33DBC) + GA4 (G-Z0M5BV38HR) |
| XLSX | SheetJS (cdn.jsdelivr.net/npm/xlsx@0.18.5) |
| Charts | Chart.js 4.4.0 |
| Tipografía | Bebas Neue + Outfit (Google Fonts) |

---

## Archivos del proyecto

```
forge-firebase.html     ← LA APP PRINCIPAL — no tocar sin leer DECISIONS.md
index.html              ← Landing page (copia de forge-landing.html)
forge-landing.html      ← Landing page fuente
forge-feedback.html     ← Formulario de feedback del piloto
forge-checklist.html    ← Checklist pre-lanzamiento
firebase-rules-pilot.json ← Reglas de seguridad Firebase
forge-business.md       ← Análisis de negocio, FODA, precios y proyecciones
```

---

## Reglas de trabajo

### ANTES de cualquier cambio
1. Leer `DECISIONS.md` — especialmente "Zonas protegidas"
2. Verificar sintaxis JS con Node antes de entregar:
```bash
# Extraer el script principal y validar con node --check (más preciso que compileFunction)
python3 -c "
content = open('forge-firebase.html', encoding='utf-8').read()
lastStart = content.rfind('<script>') + 8
lastEnd = content.rfind('</script>')
open('/tmp/forge_check.js','w').write(content[lastStart:lastEnd])
"
node --check /tmp/forge_check.js
# Sin output = OK. Con output = línea exacta del error.
```

> ⚠️ No usar `compileFunction` — no detecta todos los errores. Usar siempre `node --check`.
3. Nunca editar directamente sin haber leído el contexto del módulo

### Patrón de edición segura
```python
# Siempre usar python para ediciones — más confiable que str_replace en archivos grandes
content = open('forge-firebase.html', encoding='utf-8').read()
content = content.replace(OLD, NEW, 1)  # siempre con count=1
open('forge-firebase.html', 'w', encoding='utf-8').write(content)
```

### Firebase — reglas críticas
- El nodo `forge/logs` necesita `.read: true` a nivel colección (no solo `$logId`)
- Sin eso el entrenador no puede ver las sesiones de sus atletas
- El nodo `forge/users/$uid` necesita `auth.uid == $uid` para leer/escribir el perfil
- El archivo `firebase-rules-pilot.json` tiene las reglas correctas actualizadas

### Arrays en Firebase
Firebase convierte arrays en objetos `{0:{...}, 1:{...}}` al guardar.
Siempre usar `toArr()` y `normalizeMeso()` / `normalizeLog()` al leer de Firebase.
Estas funciones están definidas en el JS del app y son críticas.

---

## Debug
Usar el patrón `?debug=1` para panels de diagnóstico:
```javascript
var DEBUG = new URLSearchParams(window.location.search).get('debug') === '1';
function debugPanel(data) {
  if (!DEBUG) return;
  // panel solo visible con ?debug=1 en la URL
}
```
Nunca dejar debug panels hardcodeados en producción.

---

## Convenciones de código

- **Sin frameworks** — JS vanilla con `var` (no `const`/`let` — compatibilidad máxima)
- **Sin arrow functions** en funciones críticas — usar `function()` explícito
- **HTML generado por JS** — todos los módulos generan HTML como strings y hacen `el.innerHTML = html`
- **Colores** — siempre usar variables CSS (`var(--accent)`, `var(--red)`, etc.)
- **IDs de tabs** — siempre ASCII sin tildes (`session` no `sesión`, `semaforo` no `semáforo`)
- **Texto visible** — sí usar tildes y ñ correctas en strings de UI
- **Emojis en tabs** — todos los tabs de nav llevan emoji prefijo. Al agregar uno nuevo, incluir emoji consistente
- **Header de dos filas** — `.hdr-row1` (logo + badge + logout) y `.hdr-row2` (nav full-width). No mezclar elementos entre filas

---

## Roles de usuario
Hay 3 roles activos en el prototipo:
- `trainer` — entrenador: gestiona mesociclos, ve dashboard con semáforo de todos sus atletas
- `athlete` — atleta vinculado: se registra con código del entrenador, solo ve su propia sesión
- `autonomous` — atleta autónomo: crea sus propios mesociclos, ve su propio semáforo, sin entrenador

El rol autónomo usa `TRAINER_CODE` propio (se lo genera a sí mismo) y `ATHLETE_ID = code + "_self"`.
`loadAutonomousApp()` combina las vistas de trainer + athlete en una sola interfaz con 4 tabs: Plan · Sesión · Stats · Salud.

⚠️ DECISIÓN PENDIENTE (crítica para el SaaS): un usuario autónomo que quiera vincularse a un entrenador
necesita un mecanismo de roles múltiples. Hoy no está implementado. Ver DECISIONS.md.

## Autenticación
El prototipo usa Firebase Auth con email/password. El perfil se guarda en `forge/users/{uid}`.
Los datos del usuario NO están en localStorage — están en Firebase, lo que permite acceso multi-dispositivo.


---

## Patrón para agregar una pestaña nueva

Cada tab nuevo requiere exactamente 4 cambios:

1. **HTML** — `<section id="section-X" class="section"></section>` en `<main>`
2. **Nav** — botón en `loadTrainerApp()`, `loadAthleteApp()`, `loadAutonomousApp()` según corresponda
3. **`renderCurrentTab()`** — `else if (activeTab === 'X') renderX();`
4. **Triggers en listeners** — si usa LOGS, agregar `if (activeTab === 'X') renderX();` en los handlers de Firebase

Sin alguno de estos 4 pasos, la pestaña no funciona o no se actualiza en tiempo real.

---

## Sistema de tips contextuales

```javascript
// Agregar un tip en cualquier sección — se descarta con ✓ y no vuelve a aparecer
html += buildTip('id_unico', '💡', '<strong>Título</strong> Descripción.');
// IDs descriptivos: 'plan_empty', 'session_start', 'stats_first', etc.
// Persiste en localStorage bajo forge_tips_dismissed
```

## Contexto de negocio
- Target primario: entrenadores personales independientes en Argentina
- Target secundario: atletas autónomos (canal de adquisición top-of-funnel)
- Precio: Autónomo Free (3 meses) · Autónomo Pro $8k · Starter $18k · Pro $35k · Gym $75k ARS/mes
- Trial: 3 meses gratis para autónomos, 2 meses para entrenadores — sin tarjeta
- Diferencial: semáforo de autorregulación (RIR + reps) para decisiones de progresión
- Piloto activo con entrenador real — feedback en ~1 mes via `forge-feedback.html`
