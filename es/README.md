# Sistema de Prediccion de Probabilidades de Futbol

**[English Version](../README.md)** | Espanol

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

## Visualizaciones

Diagramas interactivos y explicaciones visuales del sistema. **[Ver todas las visualizaciones →](../visualizations/README.md)** *(en ingles)*

| Visualizacion | Descripcion |
|---------------|-------------|
| [Arquitectura del Sistema](../visualizations/01-system-architecture.md) | Vision de componentes y responsabilidades |
| [Pipeline de Flujo de Datos](../visualizations/02-data-flow-pipeline.md) | Transformacion de datos de extremo a extremo |
| [ML + Fallback Poisson](../visualizations/03-ml-poisson-fallback.md) | Arbol de decision de prediccion hibrida |
| [Calculo de xG Ponderado](../visualizations/04-xg-weighted-calculation.md) | Metodologia de ponderacion temporal |
| [Matriz de Puntajes Poisson](../visualizations/05-poisson-score-matrix.md) | De xG a probabilidades de partido |
| [Deteccion de Valor](../visualizations/06-value-detection.md) | Comparacion de mercado e identificacion +EV |

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
