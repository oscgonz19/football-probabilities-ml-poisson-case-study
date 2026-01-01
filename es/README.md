# Sistema de Prediccion de Probabilidades de Futbol

**[English Version](../README.md)** | Espanol

Un caso de estudio documentando la arquitectura e implementacion de un motor de prediccion de probabilidades deportivas que combina predicciones ML con metodos estadisticos tradicionales basados en Poisson.

---

## Ejemplo de Prediccion

<p align="center">
  <img src="../images/05_match_prediction.png" alt="Dashboard de Prediccion" width="800"/>
</p>

*Salida completa de prediccion: probabilidades 1X2, goles esperados (xG), BTTS y mercados Over/Under.*

---

## Resumen

Este sistema calcula probabilidades pre-partido para partidos de futbol y las compara contra cuotas del mercado para identificar oportunidades de valor esperado positivo (+EV). Presenta una arquitectura hibrida con fallback automatico—asegurando que las predicciones se completen incluso cuando los servicios ML externos no estan disponibles.

### Analisis de Distribucion de Goles

<p align="center">
  <img src="../images/01_goals_distribution.png" alt="Distribucion de Goles" width="750"/>
</p>

*Los equipos locales promedian 1.70 goles vs 1.34 para visitantes—la base de nuestro modelo Poisson.*

---

## Como Funciona

### 1. Modelado de Fortaleza de Equipos

<p align="center">
  <img src="../images/03_team_strength.png" alt="Analisis de Fortaleza" width="700"/>
</p>

Cada equipo se posiciona por fortaleza ofensiva (goles anotados) vs fortaleza defensiva (goles recibidos). El tamano indica cantidad de partidos, el color muestra diferencia de goles. Los equipos en el cuadrante inferior derecho son los mas fuertes.

### 2. Matriz de Puntajes Poisson

<p align="center">
  <img src="../images/06_probability_grid.png" alt="Matriz de Probabilidades" width="650"/>
</p>

Usando goles esperados (xG), construimos una matriz de probabilidad para cada marcador posible. El resultado mas probable (1-1 con 12.4%) ancla nuestros calculos de mercado.

### 3. Calibracion del Modelo

<p align="center">
  <img src="../images/04_calibration.png" alt="Grafico de Calibracion" width="550"/>
</p>

*Cuando predecimos 60% de probabilidad, los eventos ocurren ~60% del tiempo. Mientras mas cerca de la diagonal, mejor calibrado.*

---

## Rendimiento Historico

<p align="center">
  <img src="../images/02_results_by_season.png" alt="Resultados por Temporada" width="700"/>
</p>

Cuatro temporadas de datos muestran ventaja local consistente (~44-46% victorias locales) con patrones estables—validando nuestras suposiciones de modelado.

---

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

---

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

## Licencia

Esta documentacion de caso de estudio se proporciona para propositos de portafolio.

---

*Ningun credencial, URL interna, ni detalle de implementacion propietario esta incluido en este repositorio.*
