# FORGE — Contexto completo del proyecto

---

## Origen

La idea surgió de una necesidad real: el fundador usa un entrenador personal que gestiona
todas las rutinas en Google Sheets. La experiencia de cargar datos en una planilla desde
el celular en el gym, sin gráficos ni estadísticas, fue el pain point inicial.

**El fundador es el usuario con el problema.** Su entrenador es el primer tester del piloto.

---

## Estado actual (marzo 2026)

| Item | Estado |
|---|---|
| Prototipo HTML | ✅ Funcional con Firebase |
| Auth Firebase | ✅ Email/password, multi-dispositivo |
| 3 roles de usuario | ✅ Trainer, Athlete, Autonomous |
| Manual de usuario | ✅ Pestaña ❓ Ayuda en la app (por rol) |
| Pestaña 🏆 Progreso | ✅ Racha, semáforo, PRs, mini gráfico |
| Pestaña 📋 Historial | ✅ Sesiones expandibles con tabla serie/reps/kg/RIR |
| Plantillas de mesociclo | ✅ PPL, Full Body, Torso/Pierna, 4 días |
| Gestión de sesiones del plan | ✅ Agregar, renombrar, eliminar sesiones |
| Cancelar/retroceder en sesión | ✅ ✕ cancelar + ← volver al ejercicio anterior |
| Tips contextuales | ✅ Guías descartables por sección |
| Header dos filas | ✅ Logo+badge en fila 1, tabs full-width en fila 2 |
| Piloto | 🔄 En curso — entrenador real + atletas reales |
| Hosting | ✅ GitHub Pages (Netlify descartado por límite de plan) |
| Landing page | ✅ Publicada en GitHub Pages |
| Formulario feedback | ✅ Listo para enviar al mes |
| Análisis de negocio | ✅ FODA + precios + proyecciones en forge-business.md |
| SaaS Next.js | ⏳ Pendiente — depende del feedback del piloto |

---

## Arquitectura del prototipo

### Auth — Firebase Email/Password
- Login con email + contraseña (sin Google OAuth por incompatibilidad con GitHub Pages)
- Perfil guardado en `forge/users/{uid}` — multi-dispositivo, no depende del browser
- Al reabrir la app, `onAuthStateChanged` restaura la sesión automáticamente
- Nuevo usuario → elige rol → flujo de setup según el rol

### Roles
- `trainer` — genera código de 6 letras, gestiona atletas y mesociclos
- `athlete` — se vincula con código del entrenador, solo ve su sesión
- `autonomous` — crea su propio perfil de entrenador + atleta self-vinculado

⚠️ **Decisión pendiente**: un autónomo que quiera vincularse a un entrenador necesita
soporte de roles múltiples en el schema de usuarios. No implementado en el prototipo.
En el SaaS: `forge/users/{uid}.roles = ["autonomous", "athlete"]` con sub-objetos por rol.

### Flujo de datos
```
Entrenador (trainer)
  └── Crea mesociclos → Firebase: forge/trainers/{TRAINER_CODE}/mesocycles
  └── Ve dashboard con semáforo ← lee forge/logs (filtrado por trainerId)
  └── Asigna meso a atletas → forge/trainers/{TRAINER_CODE}/athletes/{ATHLETE_ID}

Atleta (athlete)
  └── Se vincula con código de 6 letras → queda en forge/trainers/{code}/athletes/
  └── Lee mesociclo asignado → forge/trainers/{code}/mesocycles/{assignedMesoId}
  └── Registra sesiones → forge/logs/{logId}
```

### Estructura de datos Firebase
```
forge/
  users/
    {uid}/                        ← Firebase Auth UID
      role: "trainer" | "athlete" | "autonomous"
      trainerCode: string
      athleteId: string|null
      name: string
      email: string
      createdAt: ISO string

forge/
  trainers/
    {TRAINER_CODE}/          ← código de 6 letras, ej: "BC35U6"
      name: string
      code: string
      activeMesoId: string
      mesocycles/
        {mesoId}/
          name, weeksCount, sessions: [...]
      athletes/
        {ATHLETE_ID}/        ← formato: "{trainerCode}_{nombreLowercase}"
          name, joinedAt, assignedMesoId
  logs/
    {logId}/
      trainerId, athleteId, mesocycleId
      weekNumber, sessionId, sessionName
      date, startedAt, durationMinutes
      completed: boolean
      health: { sleep, stress, energy, notes }
      exercises: [{ exerciseId, exerciseName, muscleGroup, sets: [...] }]
```

### Módulos principales (secciones del JS)

| Módulo | Función |
|---|---|
| Firebase Init | Inicializa conexión, valida config, inicializa AUTH |
| Auth | Email/password Firebase, `onAuthStateChanged`, perfil en `forge/users/{uid}` |
| Login | 3 pasos: auth → rol → setup. Roles: trainer/athlete/autonomous/demo |
| Plan | CRUD de mesociclos, importador XLSX, plantillas predefinidas, add/rename/delete sesiones |
| Dashboard | Semáforo de autorregulación por atleta |
| Session | Flujo de entrenamiento: health check → ejercicios → finish. Cancelar + retroceder |
| Stats | Gráficos de progreso (Chart.js) |
| Health | Correlación sueño/estrés/rendimiento |
| Progreso | Racha semanal, semáforo última sesión, PRs, mini gráfico (athlete + autonomous) |
| Historial | Sesiones expandibles con tabla serie/reps/kg/RIR (athlete + autonomous) |
| Ayuda | Manual contextual por rol con tips descartables |
| Tips | `buildTip()` / `dismissTip()` — guías contextuales persistidas en localStorage |

---

## Precios y trials (actualizado)

| Plan | Precio | Trial |
|---|---|---|
| Autónomo Free | $0 | 3 meses completos, luego Autónomo Pro |
| Autónomo Pro | $8.000 ARS/mes | — |
| Entrenador Starter | $18.000 ARS/mes | 2 meses gratis |
| Entrenador Pro | $35.000 ARS/mes | 2 meses gratis |
| Gym | $75.000 ARS/mes | 2 meses gratis |

El plan Free de Autónomo es el top-of-funnel: el atleta usa FORGE, progresa,
le muestra sus gráficos al entrenador → el entrenador se convierte en cliente pago.
Actualizar precios cada 3 meses por inflación, con 30 días de aviso previo.

## Semáforo de autorregulación

El diferencial técnico del producto. Lógica:

```
Para cada ejercicio de la última sesión completada:

  SI reps_alcanzadas >= reps_objetivo:
    SI RIR_promedio >= RIR_objetivo:
      → VERDE: "+2.5kg sugerido"
    SINO:
      → AMARILLO: "Mantener carga"
  SINO:
    → ROJO: "Revisar peso o descanso"
```

Implementado en `getExSemaphore(log, exLog, exDef)`.

---

## Modo demo

- Activado desde el login con botón "Ver Demo Interactivo"
- Carga datos en memoria (sin tocar Firebase)
- 3 atletas ficticios con 3 semanas de historial
- Switch trainer/atleta en la barra superior naranja
- `IS_DEMO = true` bloquea todas las escrituras a Firebase
- `exitDemo()` limpia el estado y vuelve al login

---

## Importador XLSX

Formato esperado (orden de columnas):
```
Semana | Sesion | Ejercicio | Grupo | Series | Reps | Kg | RIR | Tempo | Descanso | Indicaciones
```

- Detección flexible de headers (normaliza tildes, mayúsculas, espacios)
- Usa semana 1 como template del mesociclo
- Si hay múltiples semanas, `weeksCount` se ajusta automáticamente
- `guideWeight` se lee del campo `Kg` / `Peso`

---

## Firebase — problemas conocidos y soluciones

### Arrays → Objetos
Firebase convierte arrays en objetos al guardar. Solución: `toArr()` + `normalizeMeso()` + `normalizeLog()`.

### Queries sin índice
`orderByChild()` sin `.indexOn` en las reglas devuelve vacío. Solución: leer `forge/logs` completo y filtrar client-side.

### Reglas de seguridad (piloto)
```json
{
  "rules": {
    "forge": {
      "trainers": { "$trainerId": { ".read": true, ".write": true } },
      "logs": { ".read": true, ".write": true }
    }
  }
}
```
⚠️ Para producción necesita autenticación Firebase real.

---

## Analytics

- **GTM**: GTM-NCX33DBC
- **GA4**: G-Z0M5BV38HR
- Eventos trackeados: `login`, `session_start`, `session_complete`, `click_demo`, `demo_start`, `feedback_submit`
- Patrón: `window.trackEvent(name, params)` disponible en todos los archivos

---

## Competencia analizada

| Producto | Precio | Gap |
|---|---|---|
| TrueCoach | $57 USD / 20 clientes | No tiene mercado AR, cobra en USD |
| TrainHeroic | $45 USD / 25 clientes | Ídem |
| Everfit | $20 USD / 5 clientes | Ídem |
| ABC Evo / SocioPLUS | $15-80k ARS | Solo gestión de gym, no tracking de entrenamiento |
| Google Sheets | Gratis | Sin stats, sin semáforo, sin app móvil |

**Posición de FORGE**: tracker de entrenamiento profesional en ARS, con semáforo de autorregulación como diferencial, 3x más barato que competencia internacional.

---

## Roadmap post-piloto (si se valida)

1. **Feedback del entrenador** → ajustes al prototipo
2. **Definir MVP SaaS** basado en scores del `forge-feedback.html`
3. **Construir SaaS**: Next.js + Turso + NextAuth (mismo stack que app.estudiolucci.com.ar)
4. **Billing**: MercadoPago para suscripciones
5. **PWA nativa**: manifest + service worker para instalación en celular

---


---

## Roadmap de fases

### Fase 1 — MVP SaaS (post-piloto)
Auth real, schema Turso, billing MercadoPago, Next.js 14.

### Fase 2 — Producto
PWA offline, notificaciones push, mensajería interna entrenador↔atleta,
plantillas de mesociclos predefinidas (PPL, Full Body, Torso/Pierna).

### Fase 3 — Escala
Multi-trainer por gimnasio, roles múltiples (autónomo + atleta vinculado),
reporte PDF semanal, progresión automática sugerida por semáforo.

### Fase 4 — IA generativa de rutinas 🔭 (largo plazo)

**Contexto**: el onboarding del atleta autónomo hoy requiere saber armar una rutina.
La IA elimina esa barrera y amplía el mercado a usuarios sin conocimiento técnico.

**Flujo propuesto**:
1. **Formulario estructurado** — datos fijos al crear el perfil:
   edad, sexo, peso, altura, nivel de experiencia (principiante/intermedio/avanzado),
   frecuencia disponible (días por semana), equipamiento disponible.

2. **Conversación con IA** — sesión guiada antes de generar la rutina:
   - Objetivos de entrenamiento (hipertrofia, fuerza, resistencia, pérdida de peso)
   - Lesiones o condiciones de salud relevantes
   - Ejercicios que no puede o no quiere hacer
   - Preferencias (ej: no hacer sentadilla, preferir máquinas sobre libres)

3. **Generación de rutina** — la IA produce un mesociclo completo con:
   sesiones, ejercicios, series, reps, kg inicial, RIR, tempo y notas de técnica.
   Formato idéntico al importador XLSX — se carga directamente en el plan.

4. **Loop de progresión adaptativa**:
   semáforo da feedback → IA ajusta la rutina la semana siguiente automáticamente.
   Es un sistema de progresión continua sin intervención manual.

5. **Revisión por entrenador** (modelo híbrido):
   En el plan Gym, el entrenador puede revisar y ajustar la rutina generada por IA
   antes de activarla. El entrenador agrega valor sin crear el plan desde cero.

**Stack técnico**: Gemini 1.5 Flash (ya en roadmap de Tesorería para OCR).
Costo estimado por rutina generada: < $0.01 USD.

**Riesgo legal**: el formulario de salud se acerca a consejo médico.
Disclaimer obligatorio: *"Esta rutina es orientativa. Consultá con un médico
antes de iniciar si tenés condiciones de salud preexistentes."*
No almacenar diagnósticos médicos — solo preferencias y objetivos de entrenamiento.

**Por qué cierra el loop del negocio**:
- Amplía el mercado de autónomos a usuarios sin conocimiento técnico
- El semáforo + IA = sistema de progresión adaptativa completo
- Diferencial competitivo difícil de replicar para competidores sin el semáforo
- El entrenador humano sigue siendo el revisor — FORGE no reemplaza al coach,
  lo asiste y le ahorra tiempo

---

## Features pendientes (puntuadas en feedback form)

- Mensajería interna entrenador ↔ atleta
- Link de video YouTube por ejercicio (técnica)
- Reporte PDF semanal por atleta
- Progresión automática sugerida basada en semáforo
- Historial de PRs por ejercicio
- Biblioteca de mesociclos predefinidos (PPL, Full Body, etc.)
- Calculadora 1RM manual
- Control de volumen por músculo (MEV/MRV)
- Reminder automático de semana de descarga
- Múltiples entrenadores por gimnasio
