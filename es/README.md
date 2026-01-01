# Sistema de Predicción de Probabilidades de Fútbol

**[English Version](../README.md)** | Español

Un caso de estudio documentando la arquitectura e implementación de un motor de predicción de probabilidades deportivas que combina predicciones ML con métodos estadísticos tradicionales basados en Poisson.

---

<p align="center">
  <img src="../images/05_match_prediction.png" alt="Dashboard de Predicción" width="800"/>
</p>

<p align="center"><em>Ejemplo de salida: probabilidades 1X2, goles esperados (xG), BTTS y mercados Over/Under para un partido.</em></p>

---

## Resumen

Este sistema calcula probabilidades pre-partido para partidos de fútbol y las compara contra cuotas del mercado para identificar oportunidades de valor esperado positivo (+EV). Presenta una arquitectura híbrida con fallback automático—asegurando que las predicciones se completen incluso cuando los servicios ML externos no están disponibles.

**Características Principales:**

- **Arquitectura Híbrida ML + Estadística** — Predicciones ML primero, fallback Poisson automático
- **Tolerancia a Respuestas Parciales** — Maneja respuestas ML incompletas gracefully
- **Testing de Contrato** — Valida invariantes matemáticos (rangos de probabilidad, consistencia)
- **Detección de Valor** — Compara probabilidades calculadas contra cuotas del mercado

---

## Cómo Funciona

<table>
<tr>
<td width="50%">

### Modelado de Fortaleza de Equipos

Cada equipo se posiciona por fortaleza ofensiva (goles anotados) vs fortaleza defensiva (goles recibidos). Los equipos en el cuadrante inferior derecho son los más fuertes.

</td>
<td width="50%">

<img src="../images/03_team_strength.png" alt="Fortaleza de Equipos" width="400"/>

</td>
</tr>
<tr>
<td width="50%">

### Matriz de Puntajes Poisson

Usando goles esperados (xG), construimos una matriz de probabilidad para cada marcador posible. Cada celda muestra la probabilidad de ese resultado exacto.

</td>
<td width="50%">

<img src="../images/06_probability_grid.png" alt="Matriz de Puntajes" width="400"/>

</td>
</tr>
<tr>
<td width="50%">

### Calibración del Modelo

Cuando predecimos 60% de probabilidad, los eventos ocurren ~60% del tiempo. Mientras más cerca de la diagonal, mejor calibrado el modelo.

</td>
<td width="50%">

<img src="../images/04_calibration.png" alt="Calibración" width="400"/>

</td>
</tr>
</table>

---

## Documentación

| Documento | Descripción | Audiencia |
|-----------|-------------|-----------|
| [Caso de Estudio Principal](football-prediction-case-study.md) | Caso de estudio completo para portafolio | General |
| [Resumen Ejecutivo](case-study-executive-summary.md) | Visión general con diagrama de arquitectura | Recruiters / Managers |
| [Anexo Técnico](case-study-technical-appendix.md) | Documentación técnica detallada | Tech Leads / Ingenieros |
| [Pipeline Explicado](probability-pipeline-explained.md) | Cómo funciona el pipeline de predicción | Data Scientists / ML Engineers |
| [Fórmulas Matemáticas](probability-pipeline-formulas.md) | Distribución Poisson y derivaciones | Estadísticos / Quants |

## Diagramas Técnicos

Diagramas de arquitectura y flujo basados en Mermaid. **[Ver todos →](../visualizations/README.md)** *(en inglés)*

| Diagrama | Descripción |
|----------|-------------|
| [Arquitectura del Sistema](../visualizations/01-system-architecture.md) | Vista de componentes y responsabilidades |
| [Pipeline de Flujo de Datos](../visualizations/02-data-flow-pipeline.md) | Transformación de datos de extremo a extremo |
| [ML + Fallback Poisson](../visualizations/03-ml-poisson-fallback.md) | Árbol de decisión de predicción híbrida |
| [Matriz de Puntajes Poisson](../visualizations/05-poisson-score-matrix.md) | De xG a probabilidades de partido |

---

## Stack Tecnológico

| Componente | Tecnología |
|------------|------------|
| Backend | Ruby on Rails |
| Base de Datos | PostgreSQL |
| Cliente HTTP | Faraday (retry/backoff) |
| Testing | RSpec |
| Optimización | L-BFGS-B (calibración de finishing offset) |

---

## Proyectos Relacionados

| Proyecto | Descripción |
|----------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | El servicio de predicción ML que se integra con este sistema. Incluye modelado de goles esperados, pipelines de entrenamiento y endpoints de predicción. |

---

## Licencia

Esta documentación de caso de estudio se proporciona para propósitos de portafolio.

---

*Ningún credencial, URL interna, ni detalle de implementación propietario está incluido en este repositorio.*
