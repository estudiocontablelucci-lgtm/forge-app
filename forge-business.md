# FORGE — Análisis de negocio
> Actualizado: marzo 2026. Post-piloto con 1 entrenador + 2 atletas.

---

## 1. Contexto actual

FORGE es un tracker de entrenamiento para entrenadores personales y atletas autónomos.
Tiene un prototipo HTML funcional en piloto real con Firebase. El diferencial técnico es el
**semáforo de autorregulación** (RIR + reps): el entrenador sabe qué ajustar sin estar presente.

Tres perfiles de usuario:
- **Entrenador** — gestiona múltiples atletas, ve el dashboard con semáforo
- **Atleta vinculado** — registra sesiones guiado por su entrenador
- **Atleta autónomo** — se entrena solo, crea su propio plan, ve su propio semáforo

---

## 2. FODA

### Fortalezas

| # | Fortaleza | Impacto |
|---|---|---|
| F1 | **Semáforo de autorregulación** — diferencial técnico sin equivalente directo en el mercado local | Alto |
| F2 | **Importador XLSX** — migración sin fricción desde Google Sheets, el tool actual del entrenador | Alto |
| F3 | **Tres perfiles** — un solo producto sirve a entrenadores, atletas vinculados y autónomos | Medio |
| F4 | **Precios en ARS** — sin necesidad de tarjeta internacional ni cuenta en dólares | Alto |
| F5 | **Mobile-first, sin instalación** — PWA accesible desde cualquier browser | Medio |
| F6 | **Correlación salud-rendimiento** — datos de sueño/estrés cruzados con progreso (feature premium en competidores) | Medio |
| F7 | **Stack propio (Next.js + Turso)** — control total, sin dependencia de plataformas terceras para el SaaS | Medio |

### Debilidades

| # | Debilidad | Mitigación |
|---|---|---|
| D1 | **Prototipo HTML sin auth robusta** — un mismo browser, sin múltiples dispositivos | Resuelto en SaaS con Firebase Auth |
| D2 | **Sin notificaciones** — el entrenador no sabe cuándo el atleta entrenó en tiempo real | Roadmap Fase 2 |
| D3 | **Sin mensajería interna** — sigue dependiendo de WhatsApp para la comunicación | Roadmap Fase 2 |
| D4 | **Sin plantillas predefinidas** — el atleta autónomo tiene que construir el plan desde cero | Roadmap Fase 1 |
| D5 | **Un solo fundador-desarrollador** — velocidad de desarrollo limitada, riesgo de concentración | Mitigar con Claude Code |
| D6 | **Sin track record comercial** — todavía no hay entrenadores pagos | El piloto es para resolverlo |

### Oportunidades

| # | Oportunidad | Tamaño |
|---|---|---|
| O1 | **Mercado argentino desatendido** — TrueCoach y competidores no tienen versión en ARS ni soporte local | Grande |
| O2 | **El entrenador personal es un mercado en crecimiento** — más gente entrena con coach que hace 5 años | Grande |
| O3 | **Atleta autónomo como canal viral** — usa la app, la recomienda a su entrenador | Grande |
| O4 | **Gimnasios como cliente B2B** — un gym con 10 entrenadores paga $750k ARS/mes en plan Gym | Medio |
| O5 | **Integración con wearables** (Apple Watch, Garmin) — datos de frecuencia cardíaca + rendimiento | Futuro |
| O6 | **Expansión a otros países de habla hispana** — Chile, Colombia, México con moneda local | Futuro |

### Amenazas

| # | Amenaza | Probabilidad |
|---|---|---|
| A1 | **TrueCoach o TrainHeroic lanza versión en ARS** — tienen capital y producto maduro | Media |
| A2 | **Inflación argentina** — precios en ARS se erosionan; hay que actualizar frecuentemente | Alta |
| A3 | **El entrenador no abandona Google Sheets** — resistencia al cambio del usuario objetivo | Media |
| A4 | **Competidor local** — alguien replica el concepto con más recursos | Baja (hoy) |
| A5 | **Firebase cambia condiciones del plan gratuito** — ya pasó con otros servicios | Baja |

---

## 3. Análisis competitivo

| Producto | Precio USD/mes | En ARS equiv. | Límite atletas | Semáforo RIR | Precio local |
|---|---|---|---|---|---|
| **TrueCoach** | $57 | ~$57.000 | 20 clientes | ❌ | ❌ |
| **TrainHeroic** | $45 | ~$45.000 | 25 atletas | ❌ | ❌ |
| **Everfit** | $20 | ~$20.000 | 5 clientes | ❌ | ❌ |
| **My PT Hub** | $29 | ~$29.000 | Ilimitado | ❌ | ❌ |
| **Google Sheets** | $0 | $0 | "Ilimitado" | ❌ | ✅ |
| **FORGE Starter** | ~$18 | **$18.000** | 15 atletas | ✅ | ✅ |
| **FORGE Pro** | ~$35 | **$35.000** | 40 atletas | ✅ | ✅ |
| **FORGE Gym** | ~$75 | **$75.000** | Ilimitado | ✅ | ✅ |

**Conclusión**: FORGE es la única opción con semáforo de autorregulación, precio en ARS y soporte local.
El competidor más cercano en funcionalidad (TrueCoach) cuesta 3x más y cobra en dólares.

---

## 4. Esquema de precios

### Modelo de precios por perfil

| Plan | Precio | Trial | Target | Límite |
|---|---|---|---|---|
| **Autónomo Free** | $0 | **3 meses completos** | Atleta sin entrenador | Acceso completo durante el trial |
| **Autónomo Pro** | $8.000 ARS/mes | — | Atleta autónomo comprometido | Ilimitado |
| **Entrenador Starter** | $18.000 ARS/mes | **2 meses gratis** | Entrenador que empieza | 15 atletas |
| **Entrenador Pro** | $35.000 ARS/mes | **2 meses gratis** | Entrenador establecido | 40 atletas |
| **Gym** | $75.000 ARS/mes | **2 meses gratis** | Gimnasio con múltiples entrenadores | Ilimitado + multi-trainer |

> **Lógica del trial**: el entrenador tiene 2 meses para probarlo con atletas reales antes de comprometerse.
> Un mes es muy corto si el entrenador tiene atletas que no entrenan todas las semanas.
> El autónomo tiene 3 meses para ver progreso real — los gráficos recién se vuelven interesantes con 8+ semanas de datos.

### Lógica de precios

- **Autónomo Free** como canal de adquisición. El atleta usa FORGE, progresa, le muestra los gráficos a su entrenador, el entrenador se convierte en cliente pago.
- **El upgrade de Autónomo Pro a Entrenador** es natural cuando el atleta empieza a cobrar por entrenar a otros.
- **Gym** se vende al dueño del gimnasio como herramienta para sus entrenadores — no directamente a los trainers.
- **Todos los planes incluyen 30 días de prueba gratis** sin tarjeta.

### Actualización de precios por inflación

Actualizar cada 3 meses siguiendo el IPC oficial. Comunicar con 30 días de anticipación.
Los usuarios activos mantienen el precio anterior por 60 días después del ajuste.

---

## 5. Proyecciones financieras

### Supuestos

- Lanzamiento SaaS: julio 2026 (4 meses de desarrollo post-piloto)
- Churn mensual estimado: 5% (benchmark SaaS B2B pequeño)
- Costo de infraestructura: ~$30 USD/mes hasta 100 usuarios activos
- Adquisición: principalmente boca a boca + Instagram/TikTok de entrenadores

### Escenario conservador

| Mes | Trainers | Autónomos Pro | MRR (ARS) | Costo infra |
|---|---|---|---|---|
| Agosto 2026 | 5 | 10 | $180.000 | $30.000 |
| Octubre 2026 | 15 | 30 | $530.000 | $35.000 |
| Enero 2027 | 30 | 60 | $1.080.000 | $50.000 |
| Julio 2027 | 60 | 120 | $2.160.000 | $80.000 |

*(Trainers en plan Pro promedio $35k. Autónomos en Pro $8k.)*

### Escenario optimista (con 2 gyms en plan Gym)

| Mes | Trainers | Gyms | Autónomos | MRR (ARS) |
|---|---|---|---|---|
| Agosto 2026 | 10 | 1 | 20 | $585.000 |
| Enero 2027 | 50 | 3 | 100 | $2.875.000 |
| Julio 2027 | 100 | 5 | 200 | $5.975.000 |

### Break-even

Con los costos de infraestructura actuales (~$30 USD ≈ $30.000 ARS), el break-even
se alcanza con **solo 2 entrenadores en Starter** o **4 autónomos Pro**.
Es un negocio que genera margen desde el primer cliente pago.

---

## 6. Estrategia de adquisición

### Canal 1 — El entrenador del fundador (piloto actual)
Validación real, feedback directo, caso de estudio para el lanzamiento.

### Canal 2 — Red de entrenadores personales
Cada entrenador que usa FORGE vincula 5-15 atletas. Cada atleta es un potencial usuario autónomo
o una referencia a otro entrenador. **El NPS del entrenador es el KPI más importante.**

### Canal 3 — Comunidades de entrenamiento
Instagram y TikTok de entrenadores argentinos. Contenido mostrando el semáforo en acción
("este atleta estaba listo para subir 5kg y no lo sabía").

### Canal 4 — Atleta autónomo como top-of-funnel
Plan gratuito sin fricción. El atleta usa, progresa, le muestra los gráficos a su entrenador.
El entrenador se convierte en cliente pago.

---

## 7. Métricas clave a trackear

| Métrica | Objetivo 6 meses | Herramienta |
|---|---|---|
| Trainers activos (≥1 sesión/semana) | 30 | Firebase / GA4 |
| Atletas activos por trainer | ≥4 | Firebase |
| Sesiones completadas / atleta / mes | ≥8 | Firebase |
| Churn mensual | < 5% | Manual |
| NPS del entrenador | > 50 | forge-feedback.html |
| Conversión demo → cuenta | > 20% | GA4 `demo_start` vs `login` |

---

## 8. Caso de uso: atleta autónomo → cliente de entrenador

Un atleta autónomo que empiece a trabajar con un entrenador debe poder vincularse
**sin perder su historial** y sin crear una cuenta nueva.

**Flujo propuesto para el SaaS**:
1. El autónomo ya usa FORGE y tiene 3 meses de historial
2. Empieza a trabajar con un entrenador
3. Desde Configuración → "Vincularme a un entrenador" → ingresa código de 6 letras
4. La app muestra un selector al abrir: "¿Entrar como autónomo o con tu entrenador?"
5. El historial previo queda accesible en ambos modos

**Por qué esto importa para el negocio**:
- Es el caso de conversión más valioso: el autónomo trae su historial, el entrenador
  entra a una cuenta con datos reales ya cargados, la propuesta de valor es inmediata
- El entrenador puede ver el progreso previo del atleta sin sesión de onboarding
- Reduce la fricción de adopción para el entrenador

**Decisión técnica pendiente**: schema de roles múltiples (ver analysis.md sección 8.1).

---

## 9. Decisiones pendientes

1. **Trial resuelto** ✅ — 3 meses para autónomos, 2 meses para entrenadores. Sin tarjeta.

2. **¿Qué pasa post-trial sin pagar?** — Read-only (puede ver datos pero no iniciar sesiones)
   o bloqueo total. Pendiente definir con Agustín.

3. **¿MercadoPago desde el inicio o facturación manual?** ✅ Decisión tomada: manual
   los primeros 20 clientes. MP se implementa cuando haya tracción.

4. **Roles múltiples** — atleta autónomo que quiere vincularse a entrenador.
   Requiere decisión de arquitectura antes del SaaS (ver analysis.md sección 8.1).

5. **¿Cuándo asociar alguien de ventas?** — El fundador puede llegar a 20-30 clientes solo.
   Para escalar a 100+ se necesita alguien enfocado en B2B a gimnasios.

---


---

## 10. Visión a largo plazo — IA generativa de rutinas 🔭

### El problema que resuelve

El atleta autónomo hoy necesita saber armar una rutina para usar FORGE.
Eso limita el mercado a personas con conocimiento técnico previo.
La IA elimina esa barrera: el usuario responde preguntas y recibe un plan completo,
listo para registrar desde el día 1.

### Diferencial competitivo

| Feature | FORGE con IA | TrueCoach | TrainHeroic | App genérica de IA |
|---|---|---|---|---|
| Genera rutina con IA | ✅ | ❌ | ❌ | ✅ |
| Semáforo adapta la rutina | ✅ | ❌ | ❌ | ❌ |
| Entrenador humano revisa | ✅ | ✅ | ✅ | ❌ |
| Precio en ARS | ✅ | ❌ | ❌ | Variable |

**El diferencial real**: otras apps pueden generar rutinas con IA, pero ninguna
tiene el loop de feedback del semáforo. FORGE genera la rutina Y la adapta
automáticamente cada semana según el rendimiento real del atleta.
Eso es un sistema de progresión adaptativa — no un generador estático.

### Impacto en el modelo de negocio

**Nuevo segmento**: usuarios sin conocimiento técnico que hoy no pueden usar FORGE.
Estimación conservadora: 3x el mercado actual de atletas autónomos.

**Nuevo plan sugerido**:

| Plan | Precio | Incluye IA |
|---|---|---|
| Autónomo Free | $0 (3 meses) | ❌ — plan manual |
| Autónomo Pro | $8.000 ARS/mes | ❌ — plan manual |
| **Autónomo IA** | **$15.000 ARS/mes** | ✅ — generación + ajuste adaptativo |
| Entrenador Pro | $35.000 ARS/mes | ✅ — genera borradores para el entrenador |
| Gym | $75.000 ARS/mes | ✅ — IA + revisión por coach |

### Modelo híbrido — IA + entrenador humano

La IA no reemplaza al entrenador. En el plan Gym:
1. El atleta completa el formulario + conversación con IA
2. La IA genera el mesociclo completo
3. El entrenador lo revisa, ajusta lo que considera y lo activa
4. El entrenador cobra por su criterio profesional, no por el tiempo de armar la rutina

Esto es un argumento de venta para gimnasios: *"ofrecele a tus socios rutinas
personalizadas con IA revisadas por tu coach — sin que te tome 2 horas por cliente"*.

### Consideraciones legales

- No almacenar diagnósticos médicos — solo preferencias y objetivos
- Disclaimer obligatorio antes de generar la rutina
- La app no es un dispositivo médico — es una herramienta de entrenamiento
- En Argentina no hay regulación específica para IA en fitness (a marzo 2026)

### Stack técnico y costos

- **Modelo**: Gemini 1.5 Flash (ya integrado en otros proyectos del stack)
- **Costo por rutina generada**: < $0.01 USD
- **Con 1.000 usuarios IA**: ~$10 USD/mes en tokens — margen del 99%
- **Tiempo de implementación estimado**: 2-3 semanas post-Fase 3

---

*Actualizado el 2026-03-30. Revisar después del feedback del piloto (1 mes).*
