# Sistema de Prediccion de Probabilidades de Futbol

Un caso de estudio documentando la arquitectura e implementacion de un motor de prediccion de probabilidades deportivas que combina predicciones ML con metodos estadisticos tradicionales basados en Poisson.

## Resumen

Este sistema calcula probabilidades pre-partido para partidos de futbol y las compara contra cuotas del mercado para identificar oportunidades de valor esperado positivo (+EV). Presenta una arquitectura hibrida con fallback automatico—asegurando que las predicciones se completen incluso cuando los servicios ML externos no estan disponibles.

## Documentacion

| Documento | Descripcion | Audiencia |
|-----------|-------------|-----------|
| [Caso de Estudio Principal](football-prediction-case-study.md) | Caso de estudio completo para portafolio | General |
| [Resumen Ejecutivo](case-study-executive-summary.md) | Vision general con diagrama de arquitectura | Recruiters / Managers |
| [Anexo Tecnico](case-study-technical-appendix.md) | Documentacion tecnica detallada | Tech Leads / Ingenieros |
| [Pipeline Explicado](probability-pipeline-explained.md) | Como funciona el pipeline de prediccion | Data Scientists / ML Engineers |
| [Formulas Matematicas](probability-pipeline-formulas.md) | Distribucion Poisson y derivaciones | Estadisticos / Quants |

## Caracteristicas Principales

- **Arquitectura Hibrida ML + Estadistica**: Intenta predicciones ML primero, recurre a calculos Poisson automaticamente
- **Tolerancia a Respuestas Parciales**: Maneja respuestas ML incompletas gracefully
- **Testing de Contrato**: Valida invariantes matematicos (rangos de probabilidad, consistencia de sumas)
- **Integracion de Clima**: Considera condiciones climaticas para marcado de riesgo
- **Deteccion de Valor**: Compara probabilidades calculadas contra cuotas del mercado

## Stack Tecnologico

- Ruby on Rails
- PostgreSQL
- Faraday (cliente HTTP con retry/backoff)
- RSpec (testing)
- Optimizacion L-BFGS-B (calibracion de finishing offset)

## Arquitectura

```
Fuentes de Datos (API Deportiva, API Clima, Servicio ML)
                    │
                    ▼
            Capa de Ingesta
                    │
                    ▼
      Almacenamiento PostgreSQL
                    │
                    ▼
      Procesamiento Estadistico
                    │
                    ▼
    Pipeline de Prediccion (ML → Fallback Poisson)
                    │
                    ▼
    Salida: Probabilidades + Flags de Valor
```

## Proyectos Relacionados

| Proyecto | Descripcion |
|----------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | El servicio de prediccion ML que se integra con este sistema. Incluye modelado de goles esperados, pipelines de entrenamiento y endpoints de prediccion. |

---

*Ningun credencial, URL interna, ni detalle de implementacion propietario esta incluido en este repositorio.*
