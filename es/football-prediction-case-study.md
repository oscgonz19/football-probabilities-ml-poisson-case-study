# Sistema de Prediccion de Probabilidades de Futbol

> Caso de Estudio - Portafolio

---

## Que es este Sistema

Es un motor de prediccion de probabilidades deportivas que calcula probabilidades pre-partido para partidos de futbol y las compara contra las cuotas del mercado para identificar oportunidades de valor esperado positivo (+EV). El sistema opera como un pipeline automatizado diario a traves de multiples ligas, combinando rendimiento historico de equipos, metricas de goles esperados (xG), datos climaticos y predicciones opcionales de machine learning. Produce probabilidades para tres mercados principales—resultado del partido (1X2), ambos equipos anotan (BTTS) y total de goles (Over/Under)—mientras mantiene continuidad operativa mediante una arquitectura hibrida que no depende de ningun servicio externo individual.

---

## Mi Rol

Disene e implemente la capa de integracion entre el motor estadistico base y un servicio externo de prediccion ML, creando un sistema hibrido que permanece operativo ante fallas de servicio, timeouts y respuestas incompletas.

- **Diseno de Arquitectura Hibrida**: Construi un pipeline de prediccion que intenta predicciones ML primero y automaticamente recurre a calculos basados en Poisson cuando el servicio externo no esta disponible, tiene timeout o retorna datos parciales—asegurando que las predicciones siempre se completen.

- **Cliente del Servicio ML**: Desarrolle el cliente HTTP con payloads estructurados, manejo granular de errores (timeouts, errores de validacion, fallas de servidor), logica de reintentos con backoff exponencial, y logging contextual para debugging en produccion.

- **Tolerancia a Respuestas Parciales**: Implemente logica de validacion que acepta respuestas ML incompletas, registra warnings sin fallar, y completa probabilidades faltantes via fallback estadistico—permitiendo que el contrato del servicio ML evolucione independientemente.

- **Consolidacion de Codigo Legacy**: Lidere un refactor que unifico los calculos de probabilidades en un unico flujo de procesamiento, reduciendo complejidad arquitectonica mientras preservaba compatibilidad hacia atras mediante warnings de deprecacion.

- **Testing de Contrato y Regresion**: Cree una suite de tests validando invariantes matematicos (rangos de probabilidad, consistencia de sumas, valores xG razonables) cubriendo tanto escenarios de exito ML como de fallback, incluyendo tests end-to-end contra el servicio ML en vivo.

- **Hardening para Produccion**: Reconcilie la logica de parsing con el comportamiento real del servicio observado en staging, corrigiendo discrepancias en estructura de respuestas y mejorando diagnosticos de errores.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                     FUENTES DE DATOS                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ API Datos   │  │  API Clima  │  │ Servicio de Prediccion  │  │
│  │ Deportivos  │  │             │  │     ML (opcional)       │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     │
┌─────────────────────────────────────┐          │
│         CAPA DE INGESTA             │          │
│   Rate-limited, fetchers paralelos  │          │
└─────────────────┬───────────────────┘          │
                  │                              │
                  ▼                              │
┌─────────────────────────────────────┐          │
│   ALMACENAMIENTO (PostgreSQL)       │          │
│  Ligas, Temporadas, Partidos, Stats │          │
└─────────────────┬───────────────────┘          │
                  │                              │
                  ▼                              │
┌─────────────────────────────────────┐          │
│   PROCESAMIENTO ESTADISTICO         │          │
│  Promedios xG ponderados (3 temp.)  │          │
│  Calibracion de eficiencia          │          │
└─────────────────┬───────────────────┘          │
                  │                              │
                  ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PIPELINE DE PREDICCION                        │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐  │
│  │  Filtro   │ → │ Asignacion│ → │ ML / Stat │ → │ Deteccion │  │
│  │ Climatico │   │    xG     │   │  Hibrido  │   │  de Valor │  │
│  └───────────┘   └───────────┘   └───────────┘   └───────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          SALIDA                                 │
│    Probabilidades (1X2, BTTS, O/U)  +  Flags de Valor (+EV)     │
└─────────────────────────────────────────────────────────────────┘
```

El sistema ingesta datos de un proveedor de estadisticas deportivas y una API de clima, almacena registros normalizados en PostgreSQL, procesa estadisticas multi-temporada con ponderacion temporal, y ejecuta predicciones a traves de un pipeline hibrido. El servicio ML se llama cuando esta disponible pero nunca es una dependencia dura—el fallback estadistico asegura que cada partido se procese.

---

## Como Funciona

El pipeline transforma datos crudos de partidos en probabilidades de mercado a traves de siete etapas:

1. **Ingesta de Datos**: Obtiene partidos, estadisticas de equipos y cuotas del proveedor de datos deportivos. Disenado considerando limites de tasa de API y maneja datos parciales o retrasados gracefully.

2. **Integracion de Clima**: Recupera pronostico o clima historico para cada estadio. Partidos con condiciones extremas (viento fuerte, precipitacion intensa, temperaturas extremas) se marcan como alto riesgo para filtrado posterior.

3. **Actualizacion de Estadisticas**: Calcula promedios xG ponderados de las ultimas tres temporadas usando un esquema de pesos 65/25/10, ajustado dinamicamente por el porcentaje de completitud de la temporada actual.

4. **Asignacion de xG Pre-Partido**: Asigna goles esperados a cada equipo basado en su rendimiento ofensivo/defensivo, normalizado por contexto de liga (diferenciales local/visitante).

5. **Calculo de Probabilidades**: Solicita predicciones al servicio ML si esta disponible. En timeout, error o respuesta parcial, automaticamente recurre a calculos de distribucion Poisson con ajustes de eficiencia de finalizacion.

6. **Deteccion de Valor**: Compara probabilidades calculadas contra cuotas del mercado. Cuando las cuotas implicitas (1/probabilidad) son menores que las cuotas del mercado, marca la oportunidad como valor esperado positivo.

7. **Persistencia**: Almacena todas las probabilidades, metricas ajustadas y flags de valor para analisis y consumo posterior.

---

## Detalles Tecnicos que Importan

### Estrategia de Fallback

El pipeline de prediccion implementa degradacion graceful a traves de tres niveles:

| Nivel | Condicion | Accion |
|-------|-----------|--------|
| 1 | ML retorna respuesta completa | Usa predicciones ML |
| 2 | ML retorna respuesta parcial | Usa ML para mercados disponibles, llena gaps con Poisson |
| 3 | ML falla, timeout o no disponible | Usa Poisson para todos los mercados |

El fallback es automatico y no requiere intervencion de operador. El sistema registra que path se tomo para cada partido, habilitando monitoreo de salud del servicio ML.

### Tolerancia a Respuestas Parciales

La capa de validacion acepta respuestas ML incompletas por diseno. Si el servicio ML retorna probabilidades 1X2 pero omite BTTS, el sistema:
- Acepta los valores 1X2
- Registra un warning (no un error)
- Calcula BTTS usando el metodo estadistico
- Continua procesando

Esto permite que el servicio ML evolucione su contrato independientemente—nuevos mercados pueden agregarse o removerse sin romper el consumidor.

### Invariantes de Validacion

Todas las probabilidades de salida se validan antes de persistir:

```
Rango:     0.0 ≤ P ≤ 1.0 para todas las probabilidades
Suma:      |P(local) + P(empate) + P(visita) - 1.0| < 0.05
           |P(btts_si) + P(btts_no) - 1.0| < 0.05
           |P(over) + P(under) - 1.0| < 0.05
Sanidad:   0.0 ≤ xG < 10.0 (ningun equipo espera 10+ goles)
```

Partidos que fallan validacion se registran y excluyen de deteccion de valor.

### Testing de Contrato

La estrategia de testing sigue una piramide:

- **Tests unitarios**: Logica pura con dependencias mockeadas (corren en CI)
- **Tests de integracion**: HTTP stubbing con fixtures realistas (corren en CI)
- **Tests de regresion**: Llamadas reales al servicio ML en vivo, validando el contrato actual (corren bajo demanda, controlados por flag de entorno)

Los tests de regresion capturan drift de contrato que tests mockeados perderian—cambios en estructura de respuesta, campos nuevos, mercados removidos.

---

## Resultados

El sistema corre diariamente a traves de multiples ligas, procesando jornadas completas a traves del pipeline hibrido.

**Lo que validamos:**

- Sin dependencia dura del servicio ML—las predicciones se completan via fallback cuando el servicio externo esta degradado o no disponible
- Path estadistico de baja latencia habilita procesamiento batch sin bloquear en llamadas externas
- Validacion de invariantes captura datos malformados antes de que afecten analisis posterior
- Tests de regresion contra el servicio ML en vivo confirmaron alineacion de contrato despues de iteraciones en staging
- Monitoreo de tasa de fallback provee visibilidad sobre confiabilidad del servicio ML

**Lo que no medimos (aun):**

- Precision de prediccion vs lineas de cierre del mercado
- ROI a largo plazo en oportunidades +EV marcadas
- Comparacion de calidad de prediccion ML vs estadistica

Estos requieren recoleccion de datos longitudinal y estan planeados para evaluacion futura.

---

## Lecciones Aprendidas

**Construir fallback desde el dia uno.** La arquitectura hibrida no fue una ocurrencia tardia—fue un requisito de diseno. Esto significo que el sistema nunca se bloqueo por problemas del servicio ML durante desarrollo o staging.

**Tests de contrato capturan lo que los mocks pierden.** Tests unitarios con stubs verificaron logica, pero tests de regresion contra el servicio ML real capturaron discrepancias de estructura que habrian causado fallas en produccion.

**Tolerancia parcial habilita evolucion independiente.** Al aceptar respuestas incompletas en lugar de fallar duro, el equipo de ML pudo iterar en su API sin coordinar deployments.

**Logging contextual vale la pena.** Agregar IDs de partido, fuente de prediccion (ML vs estadistico), y tipos de error a cada mensaje de log hizo debugging de problemas de integracion significativamente mas rapido.

**Warnings de deprecacion preservan velocidad del equipo.** En lugar de cambios breaking a paths de codigo legacy, warnings permitieron que otras partes del sistema migraran a su propio ritmo.

---

## Proximos Pasos

- **Tracking de precision de prediccion**: Recolectar lineas de cierre y resultados reales para medir calibracion
- **Comparacion A/B**: Cuantificar calidad de prediccion ML vs estadistica una vez existan datos suficientes
- **Mercados adicionales**: Extender pipeline para soportar resultado exacto, handicap asiatico
- **Ingesta de cuotas en tiempo real**: Mover de batch diario a monitoreo continuo de cuotas
- **Alertas en tasa de fallback**: Disparar alertas cuando fallback ML exceda umbral

---

*Caso de estudio preparado para portafolio publico. No contiene credenciales, identificadores internos, ni detalles de implementacion propietarios.*
