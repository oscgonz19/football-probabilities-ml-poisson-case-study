# Sistema de Predicción de Probabilidades de Fútbol

**[English Version](../README.md)** | Español

Un caso de estudio documentando la arquitectura e implementación de un motor de predicción de probabilidades deportivas que combina predicciones ML con métodos estadísticos basados en Poisson.

---

## Vista General del Sistema

```mermaid
flowchart TB
    subgraph Fuentes["Fuentes de Datos"]
        API1[API Datos Deportivos]
        API2[API Clima]
        API3[Servicio ML]
    end

    subgraph Core["Sistema Central"]
        ING[Ingesta de Datos<br/>Rate Limited]
        DB[(PostgreSQL)]
        STATS[Motor Estadístico<br/>Promedios xG Ponderados]
    end

    subgraph Prediccion["Pipeline de Predicción"]
        ML_CHECK{ML Disponible?}
        ML_PATH[Predicciones ML]
        POISSON[Fallback Poisson]
        VALUE[Detección de Valor]
    end

    subgraph Salida["Salida"]
        PROB[Probabilidades<br/>1X2, BTTS, O/U]
        FLAGS[Señales +EV]
    end

    API1 & API2 --> ING --> DB --> STATS
    STATS --> ML_CHECK
    API3 -.->|Opcional| ML_CHECK
    ML_CHECK -->|Sí| ML_PATH --> VALUE
    ML_CHECK -->|No/Timeout| POISSON --> VALUE
    VALUE --> PROB & FLAGS
```

---

## Características Principales

- **Arquitectura Híbrida ML + Estadística** — Predicciones ML primero, fallback Poisson automático
- **Tolerancia a Respuestas Parciales** — Maneja respuestas ML incompletas gracefully
- **Testing de Contrato** — Valida invariantes matemáticos (rangos, consistencia)
- **Detección de Valor** — Compara probabilidades calculadas contra cuotas del mercado

---

## Flujo de Predicción

```mermaid
flowchart LR
    subgraph Entrada
        XG[Valores xG<br/>por Equipo]
    end

    subgraph Proceso
        MATRIX[Construir Matriz<br/>11x11 probabilidades]
        MARKETS[Calcular Mercados]
    end

    subgraph Mercados
        M1[1X2]
        M2[BTTS]
        M3[Over/Under]
    end

    subgraph Comparar
        IMPLIED[Cuotas Implícitas]
        MARKET[Cuotas Mercado]
        EV{Valor?}
    end

    XG --> MATRIX --> MARKETS
    MARKETS --> M1 & M2 & M3
    M1 & M2 & M3 --> IMPLIED
    IMPLIED --> EV
    MARKET --> EV
    EV -->|Sí| FLAG["Flag +EV"]
    EV -->|No| SKIP[Omitir]
```

---

## Estrategia de Fallback ML

```mermaid
flowchart TB
    START[Solicitud de Predicción] --> CHECK{Servicio ML<br/>Disponible?}
    CHECK -->|Sí| CALL[Llamar API ML<br/>timeout: 5s]
    CHECK -->|No| POISSON[Cálculo Poisson]

    CALL --> RESPONSE{Estado de<br/>Respuesta?}
    RESPONSE -->|200 OK| VALIDATE{Todos los<br/>Campos?}
    RESPONSE -->|Timeout/Error| POISSON

    VALIDATE -->|Sí| USE_ML[Usar Predicciones ML]
    VALIDATE -->|Parcial| HYBRID[ML + Poisson<br/>para campos faltantes]

    USE_ML & HYBRID & POISSON --> OUTPUT[Probabilidades Finales]

    style OUTPUT fill:#90EE90
```

---

## Documentación

| Documento | Descripción | Audiencia |
|-----------|-------------|-----------|
| [Caso de Estudio Principal](football-prediction-case-study.md) | Caso de estudio completo | General |
| [Resumen Ejecutivo](case-study-executive-summary.md) | Visión general | Recruiters / Managers |
| [Anexo Técnico](case-study-technical-appendix.md) | Documentación técnica | Tech Leads / Ingenieros |
| [Pipeline Explicado](probability-pipeline-explained.md) | Pipeline de predicción | Data Scientists / ML Engineers |
| [Fórmulas Matemáticas](probability-pipeline-formulas.md) | Derivaciones Poisson | Estadísticos / Quants |

## Diagramas Técnicos

Más diagramas Mermaid detallados. **[Ver todos →](../visualizations/README.md)** *(en inglés)*

---

## Stack Tecnológico

| Componente | Tecnología |
|------------|------------|
| Backend | Ruby on Rails |
| Base de Datos | PostgreSQL |
| Cliente HTTP | Faraday (retry/backoff) |
| Testing | RSpec |
| Optimización | L-BFGS-B |

---

## Proyectos Relacionados

| Proyecto | Descripción |
|----------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | El servicio de predicción ML que se integra con este sistema |

---

*Sin credenciales, URLs internas, ni detalles de implementación propietarios.*
