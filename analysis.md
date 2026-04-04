# FORGE — Análisis completo para migración a SaaS

> Consolidación de los reportes de 3 agentes de análisis: Datos, UX/Lógica, y Edge Cases.
> Fecha: 2026-03-29

---

## 1. Resumen ejecutivo

FORGE es un tracker de entrenamiento con un prototipo funcional en HTML single-file + Firebase Realtime Database, actualmente en piloto con un entrenador real y sus atletas. El diferencial técnico es el **semáforo de autorregulación** (verde/amarillo/rojo por ejercicio basado en reps alcanzadas vs RIR promedio), que permite al entrenador tomar decisiones de progresión sin estar presente en cada sesión.

**Lo que hay que construir**: un SaaS multi-tenant en Next.js 14 + Turso + NextAuth v4 que replique fielmente toda la funcionalidad del prototipo (plan de mesociclos, importador XLSX, flujo de sesión del atleta, dashboard con semáforo, stats con Chart.js, health check) y agregue autenticación real, billing con MercadoPago, PWA con soporte offline, y gestión multi-trainer por gimnasio.

**Lo que es crítico**: preservar la lógica exacta del semáforo (`getExSemaphore`), mantener el flujo estricto `healthCheck → workout → completeEx → finishSession`, asegurar aislamiento multi-tenant perfecto desde el día 1, y diseñar el schema SQL para que las queries del dashboard (semáforo por atleta) no generen N+1 queries. La fórmula 1RM Epley (`kg * (1 + reps/30)`) y los colores del semáforo (verde `#4ade80`, amarillo `#ff8c42`, rojo `#ff4545`) son intocables.

**Lo que puede esperar**: notificaciones push, mensajería interna entrenador↔atleta, biblioteca de mesociclos predefinidos, reporte PDF semanal, control de volumen por músculo (MEV/MRV), y la migración de datos del piloto Firebase (decisión de Agustín).

---

## 2. Schema final — Turso (libSQL / SQLite)

### 2.1 Modelo de datos actual en Firebase (reverse-engineered)

```
forge/
  trainers/{TRAINER_CODE}/          ← 6 chars de charset 32 (A-Z sin I,O + 2-9)
    name: string
    code: string
    createdAt: ISO string
    activeMesoId: string|null
    mesocycles/
      {mesoId}/                     ← generado por uid() = random(36).slice(2,8) + Date.now(36).slice(-4)
        id: string
        name: string
        createdAt: "YYYY-MM-DD"
        weeksCount: number (1-16)
        sessions: [                 ← Firebase convierte a {0:{...}, 1:{...}}
          {
            id: string
            name: string
            exercises: [
              {
                id: string
                name: string
                muscleGroup: string
                targetSets: number (1-10)
                guideReps: number (1-100)
                guideWeight: number (kg, step 0.5)  ← IMPORTADO DEL XLSX, NO es el peso real
                guideRir: number (0-5)
                tempo: string (ej: "3-1-0-0")
                restTime: string (ej: "2'")
                notes: string
              }
            ]
          }
        ]
    athletes/
      {ATHLETE_ID}/                 ← formato: "{trainerCode}_{nombreLowercase}"
                                    ⚠️ IMPORTANTE PARA MIGRACIÓN: este ID compuesto
                                    NO existe como campo separado en el atleta.
                                    En el SaaS los atletas tienen IDs propios (UUID).
                                    El script de migración debe:
                                    1. Extraer trainerCode del ATHLETE_ID (split por '_', tomar [0])
                                    2. Extraer nombre (split por '_', tomar [1..].join(' '))
                                    3. Crear athlete con nuevo UUID en Turso
                                    4. Re-mapear todos los workout_logs que referencian el ATHLETE_ID viejo
        name: string
        joinedAt: ISO string
        assignedMesoId: string|null

  logs/
    {logId}/                        ← generado por uid()
      id: string
      trainerId: string             ← TRAINER_CODE del entrenador
      athleteId: string             ← ATHLETE_ID
      mesocycleId: string
      weekNumber: number
      sessionId: string
      sessionName: string
      date: "YYYY-MM-DD"
      startedAt: number (Date.now() timestamp)
      durationMinutes: number|null
      completed: boolean
      health:
        sleep: number|null (4-9, horas)
        stress: number|null (1-5)
        energy: number|null (1-5)
        notes: string
      exercises: [                  ← Firebase convierte a objetos
        {
          exerciseId: string
          exerciseName: string
          muscleGroup: string
          sets: [
            {
              reps: number|""
              kg: number|""
              rir: number|""
            }
          ]
        }
      ]
```

**localStorage keys:**
- `forge_identity` → `{role, trainerCode, athleteId?, userName}`
- `forge_trainer_code_{normalizedName}` → trainer code persistence
- `forge_cw_{ATHLETE_ID}` → `{logId, exerciseIndex, healthDone}` (current workout state)

### 2.2 Tablas SQL

```sql
-- ═══════════════════════════════════════
-- v01_initial_schema.sql
-- FORGE SaaS — Turso/libSQL
-- ═══════════════════════════════════════

-- Organizaciones (tenant root)
CREATE TABLE organizations (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  plan TEXT NOT NULL DEFAULT 'starter' CHECK(plan IN ('starter','pro','gym')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Entrenadores
CREATE TABLE trainers (
  id TEXT PRIMARY KEY,
  org_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL UNIQUE,          -- NextAuth user ID
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'owner' CHECK(role IN ('owner','member')),
  invite_code TEXT NOT NULL UNIQUE,       -- 8 chars, reemplaza el TRAINER_CODE de 6
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Atletas
CREATE TABLE athletes (
  id TEXT PRIMARY KEY,
  trainer_id TEXT NOT NULL REFERENCES trainers(id) ON DELETE CASCADE,
  user_id TEXT,                           -- NextAuth user ID (nullable: puede no tener cuenta aún)
  name TEXT NOT NULL,
  email TEXT,
  assigned_meso_id TEXT REFERENCES mesocycles(id) ON DELETE SET NULL,  -- ⚠️ ver nota abajo
  joined_at TEXT NOT NULL DEFAULT (datetime('now')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Mesociclos (template de entrenamiento)
CREATE TABLE mesocycles (
  id TEXT PRIMARY KEY,
  trainer_id TEXT NOT NULL REFERENCES trainers(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  weeks_count INTEGER NOT NULL DEFAULT 4 CHECK(weeks_count BETWEEN 1 AND 16),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Sesiones dentro de un mesociclo (template)
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  mesocycle_id TEXT NOT NULL REFERENCES mesocycles(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Ejercicios dentro de una sesión (template/plan)
CREATE TABLE exercises (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  muscle_group TEXT NOT NULL DEFAULT '',
  target_sets INTEGER NOT NULL DEFAULT 3 CHECK(target_sets BETWEEN 1 AND 10),
  guide_reps INTEGER NOT NULL DEFAULT 10 CHECK(guide_reps BETWEEN 1 AND 100),
  guide_weight REAL NOT NULL DEFAULT 0,   -- peso de REFERENCIA del XLSX, NO el peso real
  guide_rir INTEGER NOT NULL DEFAULT 2 CHECK(guide_rir BETWEEN 0 AND 5),
  tempo TEXT NOT NULL DEFAULT '',
  rest_time TEXT NOT NULL DEFAULT '',
  notes TEXT NOT NULL DEFAULT '',
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Logs de entrenamiento (una entrada por sesión ejecutada)
CREATE TABLE workout_logs (
  id TEXT PRIMARY KEY,
  trainer_id TEXT NOT NULL REFERENCES trainers(id) ON DELETE CASCADE,
  athlete_id TEXT NOT NULL REFERENCES athletes(id) ON DELETE CASCADE,
  mesocycle_id TEXT NOT NULL REFERENCES mesocycles(id) ON DELETE RESTRICT,  -- RESTRICT: no borrar meso con logs
  session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE RESTRICT,    -- RESTRICT: no borrar session con logs
  week_number INTEGER NOT NULL,
  session_name TEXT NOT NULL,             -- snapshot: el nombre puede cambiar después
  date TEXT NOT NULL,                     -- "YYYY-MM-DD"
  started_at INTEGER,                     -- Unix timestamp ms
  duration_minutes INTEGER,
  completed INTEGER NOT NULL DEFAULT 0,   -- SQLite boolean: 0/1
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ⚠️ NOTA SOBRE BORRADO DE MESOCICLOS:
-- athletes.assigned_meso_id usa ON DELETE SET NULL (el atleta pierde la asignación si se borra el meso)
-- workout_logs.mesocycle_id usa ON DELETE RESTRICT (NO se puede borrar un meso que tiene logs)
-- Esto obliga a implementar soft-delete en mesocycles:
--   ALTER TABLE mesocycles ADD COLUMN deleted_at TEXT;
-- El meso "borrado" queda en la BD, los logs siguen válidos, el trainer ya no lo ve en su lista.
-- NUNCA borrar físicamente un mesociclo que tenga workout_logs asociados.

-- Health check vinculado al workout
CREATE TABLE health_checks (
  id TEXT PRIMARY KEY,
  workout_log_id TEXT NOT NULL UNIQUE REFERENCES workout_logs(id) ON DELETE CASCADE,
  sleep REAL,                             -- horas (4-9)
  stress INTEGER CHECK(stress BETWEEN 1 AND 5),
  energy INTEGER CHECK(energy BETWEEN 1 AND 5),
  notes TEXT NOT NULL DEFAULT '',
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Ejercicios ejecutados dentro de un log
CREATE TABLE workout_exercises (
  id TEXT PRIMARY KEY,
  workout_log_id TEXT NOT NULL REFERENCES workout_logs(id) ON DELETE CASCADE,
  exercise_id TEXT NOT NULL REFERENCES exercises(id) ON DELETE CASCADE,
  exercise_name TEXT NOT NULL,            -- snapshot
  muscle_group TEXT NOT NULL DEFAULT '',  -- snapshot
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Series individuales
CREATE TABLE workout_sets (
  id TEXT PRIMARY KEY,
  workout_exercise_id TEXT NOT NULL REFERENCES workout_exercises(id) ON DELETE CASCADE,
  set_number INTEGER NOT NULL,
  reps INTEGER,                           -- nullable: el atleta puede dejar vacío. INTEGER: nadie hace 8.5 reps
  kg REAL,                                -- nullable: step 0.5kg
  rir INTEGER,                            -- nullable: entero (0-5). RIR = Reps In Reserve, no hay decimales
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Suscripciones
CREATE TABLE subscriptions (
  id TEXT PRIMARY KEY,
  org_id TEXT NOT NULL UNIQUE REFERENCES organizations(id) ON DELETE CASCADE,
  plan TEXT NOT NULL DEFAULT 'starter' CHECK(plan IN ('starter','pro','gym')),
  status TEXT NOT NULL DEFAULT 'active' CHECK(status IN ('active','past_due','cancelled','trialing')),
  max_athletes INTEGER NOT NULL DEFAULT 15,  -- starter:15, pro:40, gym:9999
  current_period_start TEXT,
  current_period_end TEXT,
  mp_subscription_id TEXT,                -- MercadoPago subscription ID
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.3 Índices críticos

```sql
-- ═══════════════════════════════════════
-- v02_indexes.sql
-- ═══════════════════════════════════════

-- Dashboard del entrenador: leer todos los logs de sus atletas
CREATE INDEX idx_workout_logs_trainer_id ON workout_logs(trainer_id);

-- Filtrar logs por atleta (vista detalle del atleta)
CREATE INDEX idx_workout_logs_athlete_id ON workout_logs(athlete_id);

-- Semáforo: última sesión completada por atleta por mesociclo
CREATE INDEX idx_workout_logs_semaphore ON workout_logs(athlete_id, mesocycle_id, completed, date DESC);

-- Verificar si una sesión específica ya fue completada
CREATE INDEX idx_workout_logs_session_check ON workout_logs(mesocycle_id, week_number, session_id, athlete_id, completed);

-- Ejercicios de un workout
CREATE INDEX idx_workout_exercises_log ON workout_exercises(workout_log_id);

-- Sets de un ejercicio
CREATE INDEX idx_workout_sets_exercise ON workout_sets(workout_exercise_id);

-- Sesiones dentro de mesociclo (ordenadas)
CREATE INDEX idx_sessions_meso ON sessions(mesocycle_id, sort_order);

-- Ejercicios dentro de sesión (ordenados)
CREATE INDEX idx_exercises_session ON exercises(session_id, sort_order);

-- Atletas por trainer
CREATE INDEX idx_athletes_trainer ON athletes(trainer_id);

-- Trainers por org
CREATE INDEX idx_trainers_org ON trainers(org_id);

-- Health check por workout
CREATE INDEX idx_health_workout ON health_checks(workout_log_id);

-- Buscar trainer por invite code (login del atleta)
CREATE INDEX idx_trainers_invite_code ON trainers(invite_code);
```

### 2.4 Análisis N+1 del dashboard

**El problema**: El dashboard del entrenador muestra un semáforo por ejercicio para cada atleta. Sin optimización:

```
1 query: obtener atletas del trainer                     → N atletas
N queries: obtener último log completado por atleta      → N logs
N queries: obtener ejercicios de cada log                → N * M ejercicios
N queries: obtener sets de cada ejercicio                → N * M * S sets
N queries: obtener definición del ejercicio (para guide) → N * M lookups
```

**Solución con query consolidada**:

```sql
-- Un solo query para el semáforo de TODOS los atletas
WITH latest_logs AS (
  SELECT wl.*,
    ROW_NUMBER() OVER (
      PARTITION BY wl.athlete_id
      ORDER BY wl.date DESC, wl.created_at DESC
    ) AS rn
  FROM workout_logs wl
  WHERE wl.trainer_id = ?
    AND wl.completed = 1
)
SELECT
  a.id AS athlete_id,
  a.name AS athlete_name,
  a.assigned_meso_id,
  ll.id AS log_id,
  ll.date AS log_date,
  ll.week_number,
  ll.session_name,
  ll.session_id,
  ll.duration_minutes,
  hc.sleep, hc.stress, hc.energy,
  we.exercise_id,
  we.exercise_name,
  we.muscle_group AS we_muscle_group,
  e.guide_reps,
  e.guide_rir,
  e.guide_weight,
  ws.set_number,
  ws.reps,
  ws.kg,
  ws.rir AS set_rir
FROM athletes a
LEFT JOIN latest_logs ll ON ll.athlete_id = a.id AND ll.rn = 1
LEFT JOIN health_checks hc ON hc.workout_log_id = ll.id
LEFT JOIN workout_exercises we ON we.workout_log_id = ll.id
LEFT JOIN exercises e ON e.id = we.exercise_id
LEFT JOIN workout_sets ws ON ws.workout_exercise_id = we.id
WHERE a.trainer_id = ?
ORDER BY a.name, we.sort_order, ws.set_number;
```

Este query devuelve todos los datos del dashboard en **una sola consulta**. Se procesa en el servidor para construir la estructura del semáforo antes de enviar al cliente.

---

## 3. Arquitectura de componentes — Next.js 14

### 3.1 Mapa de rutas

| Ruta | Componente | Rol | Datos |
|---|---|---|---|
| `/` | Landing page | Público | Estático (ya existe como `index.html`) |
| `/auth/login` | NextAuth login | Público | — |
| `/auth/signup` | Registro + invite code | Público | — |
| `/dashboard` | TrainerDashboard | Trainer | Atletas + semáforo (query consolidada) |
| `/dashboard/athletes/[athleteId]` | AthleteDetail | Trainer | Tabs: semáforo, stats, salud, historial |
| `/plan` | MesocyclePlan | Trainer | CRUD mesociclos, XLSX import |
| `/plan/[mesoId]` | MesocycleEditor | Trainer | Edición de sesiones/ejercicios |
| `/session` | AthleteSession | Athlete | Picker de semana/sesión + workout |
| `/session/workout` | WorkoutFlow | Athlete | Health → ejercicios → finish |
| `/stats` | Stats | Ambos | Charts (Chart.js, `'use client'`) |
| `/health` | HealthDashboard | Athlete | Correlación sueño/estrés |
| `/settings` | Settings | Ambos | Perfil, plan, billing |
| `/settings/billing` | Billing | Trainer | MercadoPago integration |

### 3.2 Server Actions

```typescript
// ── Plan ──
'use server'
async function createMesocycle(data: CreateMesoInput): Promise<Mesocycle>
  // Validar: trainer autenticado, límite de mesociclos si aplica
async function updateMesocycle(mesoId: string, data: UpdateMesoInput): Promise<void>
  // Validar: mesociclo pertenece al trainer
async function deleteMesocycle(mesoId: string): Promise<void>
  // Validar: no tiene logs activos (o soft delete)
async function cloneMesocycle(mesoId: string): Promise<Mesocycle>
async function importXLSX(formData: FormData): Promise<Mesocycle>
  // SheetJS en Node runtime, validar tamaño < 5MB, sanitizar

// ── Ejercicios ──
async function addExercise(sessionId: string, data: ExerciseInput): Promise<void>
  // Validar: session pertenece a mesociclo del trainer
async function updateExercise(exerciseId: string, data: Partial<ExerciseInput>): Promise<void>
async function deleteExercise(exerciseId: string): Promise<void>
  // Undo: soft delete con TTL de 10 segundos

// ── Asignación ──
async function assignMesocycle(athleteId: string, mesoId: string | null): Promise<void>
  // Validar: atleta pertenece al trainer, meso pertenece al trainer

// ── Sesión de entrenamiento ──
async function startWorkoutSession(data: StartSessionInput): Promise<WorkoutLog>
  // Crear workout_log + workout_exercises + workout_sets (vacíos)
  // Validar: atleta tiene meso asignado, sesión existe en meso
async function saveSetData(setId: string, data: SetInput): Promise<void>
  // Llamado frecuentemente — debe ser rápido
  // Optimistic update en el cliente
async function saveHealthCheck(logId: string, data: HealthInput): Promise<void>
  // Validar: log pertenece al atleta autenticado
async function completeWorkout(logId: string): Promise<void>
  // Marcar completed=1, calcular duration_minutes
  // Validar: log pertenece al atleta, no ya completado
```

### 3.3 Componentes clave

```
components/
├── semaphore/
│   ├── SemaphoreRow.tsx          ← verde/amarillo/rojo por ejercicio
│   └── SemaphoreGrid.tsx         ← grid completo de un log
├── workout/
│   ├── GuideBand.tsx             ← banda: series | reps obj | peso guía | RIR | 1RM est
│   ├── SessionClock.tsx          ← cronómetro de sesión (client-only)
│   ├── RestTimer.tsx             ← timer flotante 1/2/3 min, rojo al expirar
│   ├── ObsBanner.tsx             ← banner naranja de indicaciones del entrenador
│   ├── WorkoutProgress.tsx       ← barra + ejercicios restantes
│   ├── SetInputRow.tsx           ← fila de input: reps | kg | rir
│   └── FinishScreen.tsx          ← pantalla de celebración + confetti
├── health/
│   ├── HealthCheck.tsx           ← check de bienestar (sueño, estrés, energía)
│   └── HealthCharts.tsx          ← gráficos de correlación (client-only)
├── plan/
│   ├── XLSXImporter.tsx          ← drag & drop + preview tabla
│   ├── MesocycleCard.tsx         ← card de mesociclo con sesiones colapsables
│   └── ExerciseRow.tsx           ← fila editable de ejercicio
├── dashboard/
│   ├── AthleteCard.tsx           ← card del atleta con semáforo inline
│   └── InviteCodeBox.tsx         ← código de invitación copiable
├── stats/
│   ├── StatsCharts.tsx           ← 1RM, peso max, tonelaje, sesiones/mes (client-only)
│   └── StatSummary.tsx           ← boxes de resumen
└── ui/
    ├── Toast.tsx                 ← toast con undo
    ├── Modal.tsx                 ← modal reutilizable
    └── DebugPanel.tsx            ← panel ?debug=1
```

### 3.4 Flujo de autenticación

```
Trainer:
  1. Signup con email/password (NextAuth Credentials o Google OAuth)
  2. Se crea organization + trainer + subscription(starter, trialing)
  3. Se genera invite_code de 8 chars (charset 32, ~40 bits de entropía)
  4. Dashboard muestra el código para compartir con atletas

Athlete:
  1. Abre /auth/signup?code=XXXXXXXX (o ingresa el código manualmente)
  2. Signup con email/password o Google OAuth
  3. Se vincula al trainer automáticamente
  4. Sesión persistente: JWT con refresh token, 30 días de duración
  5. En el gym: el atleta abre la PWA, ya está logueado

Gym admin (plan Gym):
  1. Mismo signup que trainer
  2. Puede invitar otros trainers a su organización
  3. Cada trainer gestiona sus propios atletas
  4. El admin ve stats agregadas de toda la org
```

### 3.5 Estrategia PWA / Offline

**Cacheo (service worker):**
- App shell (HTML/CSS/JS) — cache first
- Mesociclo asignado — stale while revalidate
- Últimos 5 logs del atleta — stale while revalidate

**Offline recording:**
- El atleta puede iniciar y completar una sesión completa sin conexión
- Los datos se guardan en IndexedDB como cola de sincronización
- Al recuperar conexión: sync automático con conflict resolution (last-write-wins por set)
- Indicador visual: badge "offline" + "X cambios pendientes"

**Limitaciones offline:**
- No puede ver logs de otros atletas (trainer)
- No puede importar XLSX (requiere server action)
- No puede cambiar mesociclo asignado

---

## 4. Riesgos y edge cases

### 4.1 Críticos (bloquean lanzamiento)

| # | Riesgo | Descripción | Mitigación |
|---|---|---|---|
| C1 | **Aislamiento multi-tenant** | Un trainer no debe ver datos de otro trainer. Un atleta no debe ver datos de otros atletas. | Verificar `trainer_id` en CADA server action. Tests de aislamiento obligatorios antes de lanzar. |
| C2 | **IDOR en rutas dinámicas** | `/dashboard/athletes/[athleteId]` sin verificación de ownership permite acceder a datos de otros trainers. | Middleware de autorización: cada server action verifica que el recurso pertenece al usuario autenticado. |
| C3 | **Sesión interrumpida** | Atleta cierra el browser a mitad de sesión → pierde el progreso. | Persistir `currentWorkout` en IndexedDB (equivalente al `forge_cw_*` de localStorage). Al reabrir la app, restaurar automáticamente. Guardar cada `saveSetData` individualmente. |
| C4 | **Race condition en startSession** | El prototipo resuelve con `LOGS[logId] = log` local inmediato. En SaaS necesita Optimistic Update. | `useOptimistic` de React 19 + `startTransition`. El server action crea el log y devuelve el ID. Si falla, revert local. |
| C5 | **Semáforo con datos null** | Si `assignedMeso.sessions` es null o las exercises llegan vacías, el dashboard crashea. | Null checks en `getSemaphore()`. El prototipo falla silenciosamente (muestra vacío); el SaaS debe mostrar "Sin datos" explícitamente. |

### 4.2 Importantes (resolver antes de onboarding masivo)

| # | Riesgo | Descripción | Mitigación |
|---|---|---|---|
| I1 | **Código de invitación adivinable** | 6 chars de charset 32 = 32^6 = ~1 billón de combinaciones (~30 bits de entropía). Brute force viable con rate limiting bajo. | Subir a 8 chars (32^8 = ~40 bits). Rate limiting: 5 intentos/min por IP. Lockout temporal después de 10 fallos. |
| I2 | **XLSX malicioso** | Fórmulas de Excel pueden ejecutar código. Archivos gigantes pueden causar OOM. | Validar: máx 5MB, máx 500 filas, sanitizar celdas (strip fórmulas que empiezan con `=`, `+`, `-`, `@`). Procesar en server action, no en edge runtime. |
| I3 | **Trainer cambia meso durante sesión activa** | Si el trainer reasigna un meso mientras el atleta tiene un workout activo, los exercise IDs del log ya no matchean con el nuevo meso. | El log guarda snapshot de exercise_id + exercise_name. El cambio de meso no afecta sesiones en curso. Mostrar warning al trainer: "Este atleta tiene una sesión activa". |
| I4 | **Dos dispositivos simultáneos** | Mismo atleta abre la app en dos tabs/dispositivos. Ambos escriben sets al mismo log. | `workout_sets` usa `(workout_exercise_id, set_number)` como unique constraint. Last-write-wins. Mostrar warning: "Sesión activa en otro dispositivo". |
| I5 | **Chart.js en SSR** | Chart.js es client-only. Si se importa en un Server Component, crashea. | Todos los componentes con Chart.js marcados con `'use client'`. Dynamic import con `{ ssr: false }`. |
| I6 | **SheetJS en server action** | SheetJS funciona en Node pero no en Edge Runtime. | Forzar `runtime = 'nodejs'` en la route que usa SheetJS. Importar SheetJS dinámicamente solo en el server action. |

### 4.3 Menores (resolver post-MVP)

| # | Riesgo | Descripción | Mitigación |
|---|---|---|---|
| M1 | **Atleta completa más sesiones que weeksCount** | El prototipo no lo impide. ¿Es un bug o feature? | Mostrar warning "Has completado todas las semanas del mesociclo" pero permitir continuar. El entrenador decide si reasignar. |
| M2 | **Trainer elimina ejercicio en progreso** | El atleta tiene un workout con un ejercicio que el trainer acaba de eliminar del meso. | El workout_exercise ya tiene el snapshot. No se afecta. Pero el semáforo para ese ejercicio ya no encuentra `exDef` → mostrar "gray" sin sugerencia. |
| M3 | **Timezone Argentina** | `new Date().toISOString().slice(0,10)` puede dar el día incorrecto dependiendo del timezone del browser. | Usar `date` en UTC-3 explícitamente, o permitir al usuario configurar su timezone. |

### 4.4 Límites de suscripción — Edge cases

| Escenario | Comportamiento recomendado | ¿Consultar a Agustín? |
|---|---|---|
| Starter (15 atletas) intenta agregar #16 | Bloquear con modal: "Tu plan permite hasta 15 atletas. Upgrade a Pro para tener hasta 40." | **Sí** — definir UX del upsell |
| Plan expira durante sesión activa | NO interrumpir la sesión. Permitir completar el workout. Bloquear iniciar nuevas sesiones después. | **Sí** — grace period? |
| Downgrade Pro→Starter con 30 atletas | NO eliminar datos. Bloquear agregar nuevos atletas. Los existentes siguen accediendo. | **Sí** — ¿cuáles atletas quedan "activos"? |
| Trial termina sin pagar | Read-only mode: puede ver datos pero no crear sesiones ni mesociclos. | **Sí** — duración del trial y comportamiento |

---

## 5. Tests críticos pre-lanzamiento

### 5.1 Semáforo

```typescript
describe('getExSemaphore', () => {
  it('returns GREEN when reps >= guideReps AND avgRir >= guideRir', () => {
    // Sets: [{reps:10, kg:80, rir:3}, {reps:10, kg:80, rir:2}]
    // guideReps:10, guideRir:2 → avgRir=2.5 >= 2, reps OK → GREEN "+2.5kg"
  });

  it('returns YELLOW when reps >= guideReps AND avgRir < guideRir', () => {
    // Sets: [{reps:10, kg:80, rir:1}, {reps:10, kg:80, rir:1}]
    // guideReps:10, guideRir:2 → avgRir=1 < 2, reps OK → YELLOW "Mantener"
  });

  it('returns RED when any set has reps < guideReps', () => {
    // Sets: [{reps:8, kg:80, rir:3}, {reps:10, kg:80, rir:3}]
    // guideReps:10 → repsOk=false (8 < 10) → RED "Revisar peso"
    // NOTA: usa .every(), no .some() — TODOS los sets deben alcanzar las reps
  });

  it('returns GRAY when no valid sets', () => {
    // Sets: [{reps:'', kg:'', rir:''}]
    // → GRAY "Sin datos"
  });
});
```

### 5.2 Aislamiento multi-tenant

```typescript
describe('Multi-tenant isolation', () => {
  it('trainer A cannot access trainer B athletes via server action');
  it('trainer A cannot read workout_logs with trainer_id != own');
  it('athlete cannot access other athlete workout via direct URL');
  it('IDOR: /dashboard/athletes/[other_trainer_athlete_id] returns 403');
});
```

### 5.3 Importador XLSX

```typescript
describe('XLSX Import', () => {
  it('imports standard format: Semana|Sesion|Ejercicio|Grupo|Series|Reps|Kg|RIR|Tempo|Descanso|Indicaciones');
  it('handles missing optional columns (Kg, Tempo, Descanso, Indicaciones)');
  it('rejects file > 5MB');
  it('rejects file with > 500 rows');
  it('sanitizes cells starting with =, +, -, @');
  it('handles accented headers (Sesión, Ejercicio, etc.)');
  it('uses week 1 as template, sets weeksCount from max week');
});
```

### 5.4 Flujo de sesión

```typescript
describe('Workout flow', () => {
  it('follows strict order: healthCheck → exercises → finish');
  it('restores session after browser close (IndexedDB)');
  it('saves each set individually (not batch)');
  it('calculates durationMinutes on completion');
  it('shows confetti and summary on finish');
});
```

---

## 6. Decisiones pendientes — Input de Agustín requerido

### Alta prioridad (bloquean implementación)

1. **Migración de datos del piloto Firebase → Turso**: ¿Migrar los datos reales del entrenador y sus atletas, o arrancar limpio? Si se migra, se necesita un script de transformación. Si no, ¿el entrenador pierde el historial?

2. **Onboarding del trainer**: ¿Cuál es el flujo ideal para la primera vez? ¿Wizard de 3 pasos (crear perfil → importar meso → invitar atleta)? Esto es clave para conversión.

3. **Duración del trial**: ¿Cuántos días de prueba gratuita? ¿Qué features están disponibles durante el trial?

4. **Comportamiento de plan expirado**: ¿Read-only, grace period de X días, o bloqueo total?

5. **Downgrade con exceso de atletas**: ¿Cuáles quedan activos? ¿El trainer elige, o se bloquean los más recientes?

### Media prioridad (pueden definirse durante desarrollo)

6. **Notificaciones push**: ¿Se implementan en el MVP o se dejan para Fase 2? El entrenador quiere saber cuándo el atleta completa una sesión.

7. **MercadoPago**: ¿Ya hay cuenta y API keys? ¿Se usa el SDK de suscripciones recurrentes de MP?

8. **Billing integration**: ¿Mercado Pago soporta los 3 planes con cambio de plan mid-cycle? ¿O se usa un intermediario como Stripe?

### Baja prioridad (post-MVP)

9. **Mensajería interna**: ¿Chat en tiempo real o solo notas/comentarios en sesiones?

10. **Biblioteca de mesociclos predefinidos**: ¿Quién los crea? ¿Son globales o por trainer?

11. **Reporte PDF**: ¿Generado en el server o client-side?

---

*Documento generado el 2026-03-29. Actualizar al resolver cada decisión pendiente.*

---

## 7. Correcciones post-análisis
> Aplicadas por revisión humana sobre el output de los agentes. Claude Code y Cursor deben respetar estas
> decisiones sin revertirlas.

### C-01 — `workout_sets.reps` es `INTEGER`, no `REAL`
**Qué se cambió**: el tipo del campo `reps` en `workout_sets` de `REAL` a `INTEGER`.

**Por qué**: nadie hace 8.5 repeticiones ni tiene 1.5 repeticiones en reserva. Usar `REAL` en `reps` y `rir`
es semánticamente incorrecto y puede introducir errores de comparación en la lógica del semáforo
(`reps >= guideReps`, `avgRir >= guideRir`). Solo `kg` es `REAL` porque admite decimales (82.5kg).

**Impacto**: la función `getExSemaphore` compara `repsAlcanzadas >= exDef.guideReps`. Si `reps` fuera `REAL`
por algún bug de parseo podría llegar `10.0` vs `10` y la comparación daría distinto según el motor. Con
`INTEGER` esto es imposible.

---

### C-02 — Borrado de mesociclos: `ON DELETE RESTRICT` + soft-delete obligatorio
**Qué se cambió**:
- `workout_logs.mesocycle_id` cambió de `ON DELETE CASCADE` a `ON DELETE RESTRICT`
- `workout_logs.session_id` cambió de `ON DELETE CASCADE` a `ON DELETE RESTRICT`
- Se agrega columna `deleted_at TEXT` a `mesocycles` (soft-delete)

**Por qué**: el schema original tenía `athletes.assigned_meso_id ON DELETE SET NULL` (correcto: el atleta pierde
la asignación si se borra el meso) pero `workout_logs.mesocycle_id ON DELETE CASCADE` (incorrecto: borraba
todos los logs históricos del atleta si el entrenador borraba el meso). Un entrenador que borra un mesociclo
no tiene intención de borrar 3 meses de historial de entrenamiento.

**Implementación obligatoria**:
```sql
-- v03_soft_delete_mesocycles.sql
ALTER TABLE mesocycles ADD COLUMN deleted_at TEXT;  -- NULL = activo, timestamp = borrado

-- v03_soft_delete_sessions.sql  
ALTER TABLE sessions ADD COLUMN deleted_at TEXT;
```

**Regla para Claude Code**: nunca hacer `DELETE FROM mesocycles`. Siempre:
```typescript
// ✅ Correcto
await db.run('UPDATE mesocycles SET deleted_at = ? WHERE id = ?', [new Date().toISOString(), mesoId])

// ❌ Incorrecto — bloqueado por ON DELETE RESTRICT si hay logs
await db.run('DELETE FROM mesocycles WHERE id = ?', [mesoId])
```

Filtrar siempre con `WHERE deleted_at IS NULL` en las queries del trainer.

---

### C-03 — Migración Firebase → Turso: ATHLETE_ID compuesto requiere transformación
**Qué se cambió**: documentación del formato del `ATHLETE_ID` en la sección 2.1, con el algoritmo de
transformación.

**Por qué**: en Firebase el `ATHLETE_ID` tiene formato `{TRAINER_CODE}_{nombre}` (ej: `BC35U6_agustin`).
Este ID compuesto no es un campo explícito — se construye en el frontend al hacer join. En el SaaS, los
atletas tienen IDs propios (CUID o UUID). Sin esta transformación, el script de migración asignaría IDs
incorrectos y rompería las referencias en `workout_logs.athlete_id`.

**Algoritmo de migración para el script**:
```typescript
// Para cada atleta en Firebase:
function migrateAthleteId(firebaseAthleteId: string) {
  const parts = firebaseAthleteId.split('_')
  const trainerCode = parts[0]                    // "BC35U6"
  const name = parts.slice(1).join(' ')           // "agustin" → capitalizar
  const newId = createId()                         // CUID para Turso
  return { newId, trainerCode, name }
}

// Después de migrar atletas, actualizar todas las referencias:
// workout_logs.athlete_id → nuevo UUID del atleta
```

**Decisión pendiente con Agustín**: ¿se migran los datos del piloto o se arranca limpio? Ver sección 6,
punto 1. Esta decisión debe tomarse ANTES de escribir el script de migración.

---


---

## 8. Decisiones de arquitectura pendientes — Input de Agustín requerido

### 8.1 Roles múltiples por usuario ⚠️ CRÍTICA para el SaaS

**Contexto**: un atleta autónomo que empiece a trabajar con un entrenador debería poder vincularse
sin perder su historial ni crear una cuenta nueva. Hoy el schema tiene un solo `role` por `uid`.

**Decisión requerida**: elegir entre dos opciones antes de escribir el schema SQL.

**Opción A — Roles múltiples en el perfil** (recomendada para el SaaS):
```typescript
// forge/users/{uid}
{
  roles: ["autonomous", "athlete"],
  autonomous: { trainerCode: "ABC123", athleteId: "ABC123_self" },
  athlete: { trainerCode: "XYZ789", athleteId: "XYZ789_uid8chars" }
}
```
Al abrir la app con dos roles activos, se muestra un selector:
`[ 🎯 Mi entrenamiento personal ]  [ 🤸 Con mi entrenador (Juan García) ]`

**Opción B — Vinculación desde dentro de la app** (más rápida de implementar):
El autónomo ya adentro va a Configuración → "Vincularme a un entrenador" → ingresa código.
El selector de modo aparece en el header. Sin tocar el login.

**Impacto en schema SQL**:
- Opción A requiere tabla `user_roles` many-to-many
- Opción B requiere columna `linked_trainer_id` nullable en el perfil del autónomo

**Consultar a Agustín antes de implementar el schema de usuarios.**

---

### 8.2 Período de trial y comportamiento post-trial

**Decisión tomada** (marzo 2026):
- Autónomo: **3 meses gratis** completos → luego pasa a Autónomo Pro ($8.000 ARS/mes) o queda en read-only
- Entrenador: **2 meses gratis** completos → luego pasa al plan correspondiente o bloqueo de nuevas sesiones

**Pendiente definir**:
- ¿Read-only o bloqueo total post-trial?
- ¿Grace period de X días antes del bloqueo?
- ¿El entrenador puede ver los datos de sus atletas aunque no pague?

---

### 8.3 MercadoPago vs facturación manual

**Decisión tomada**: los primeros 20 clientes se facturan manualmente (transferencia bancaria).
MP Subscriptions se implementa cuando haya tracción validada.

**Impacto**: el campo `mp_subscription_id` en la tabla `subscriptions` puede ser nullable
hasta que se implemente MP.

*Actualizado el 2026-03-30. Documento base para Claude Code y Cursor.*

