# Pipeline de Calculo de Probabilidades

> Explicacion Tecnica para Data Scientists e Ingenieros ML

---

## Vision General

Este documento explica como estadisticas crudas de equipos se transforman en probabilidades de partido a traves de un pipeline de cinco etapas. Cada etapa agrega una capa de ajuste que mejora la precision predictiva. El sistema usa Expected Goals (xG) como su metrica central y distribucion Poisson para calculos de probabilidad.

---

## Etapa 1: Obtencion de xG Base

**Que hace**: Recupera Expected Goals (xG) historicos para cada equipo, diferenciado por rendimiento como local y visitante.

**Por que importa**: xG captura la calidad de oportunidades de gol creadas/concedidas, haciendolo mejor predictor que goles reales (menos ruido por varianza).

**Datos usados**:
- xG ofensivo (goles esperados a anotar) — separado por local/visitante
- xG defensivo (goles esperados a conceder) — separado por local/visitante

**Ponderacion temporal**: Usa un promedio ponderado de las ultimas 3 temporadas:

| Temporada | Peso |
|-----------|------|
| Actual | 65% |
| Anterior | 25% |
| Dos anteriores | 10% |

El peso de la temporada actual se ajusta dinamicamente por porcentaje de completitud. Si solo se ha jugado 50% de los partidos, la influencia de la temporada actual se reduce a la mitad, dando mas peso a datos historicos.

**Analogia ML**: Similar a como podriamos ponderar datos de entrenamiento recientes mas fuertemente en aprendizaje online, pero con decaimiento de recencia explicito.

---

## Etapa 2: Ajuste por Contexto de Liga

**Que hace**: Normaliza el xG de cada equipo considerando las caracteristicas ofensivas/defensivas de su liga.

**Por que importa**: Las ligas tienen diferentes tasas promedio de goles. Un equipo con xG 1.5 en una liga defensiva no es equivalente a 1.5 xG en una liga ofensiva.

**Mecanismo**:
1. Calcular el ratio xG local/visitante de la liga
2. Usar este ratio para balancear la expectativa ofensiva del equipo con la debilidad defensiva del oponente
3. Promediar ambas perspectivas para el xG ajustado final

**Intuicion**: Conceptualmente similar a estandarizar features por su distribucion en preprocesamiento ML—aqui normalizamos por el "baseline" de la liga.

**Ejemplo**:
- Equipo A tiene xG 1.8 (ofensivo) en una liga donde equipos locales promedian 1.6 xG
- Equipo B tiene xG 1.8 en una liga donde equipos locales promedian 1.2 xG
- Despues del ajuste, el xG efectivo del Equipo A sera menor relativo a su contexto de liga

---

## Etapa 3: Offset de Eficiencia de Finalizacion

**Que hace**: Ajusta xG basado en la eficiencia historica de cada equipo para convertir oportunidades en goles.

**Por que importa**: Algunos equipos consistentemente superan su xG (buenos finalizadores), otros subperforman. Ignorar esto introduce sesgo sistematico.

**Mecanismo**:
1. Calcular un coeficiente (alpha) por equipo usando Maximum Likelihood Estimation sobre sus partidos historicos
2. Aplicar este coeficiente como multiplicador exponencial al xG

**Regularizacion (Partial Pooling)**:
- Equipos con pocos partidos tienen su offset "encogido" hacia cero (prior neutral)
- Mas partidos = mas confianza en el offset empirico
- Esto previene overfitting en equipos con datos limitados

**Analogia ML**: Conceptualmente similar a efectos aleatorios en modelos mixtos, o como inferencia Bayesiana encoge estimados hacia el prior con evidencia limitada. La formula usa:

```
alpha_final = alpha_raw × (n_partidos / (n_partidos + k))
```

Donde `k` es un hiperparametro (tipicamente 15) controlando la fuerza del shrinkage.

---

## Etapa 4: Calculo de Probabilidad Poisson

**Que hace**: Transforma valores xG ajustados en distribuciones de probabilidad de goles, luego en probabilidades de mercado.

**Por que importa**: La distribucion Poisson es el estandar de la industria para modelar eventos de conteo (goles) con baja frecuencia y alta varianza.

**Mecanismo**:
1. El xG final de cada equipo se convierte en el parametro lambda de su distribucion Poisson
2. Calcular probabilidad de anotar 0, 1, 2, ... 10 goles para cada equipo
3. Construir una matriz de marcadores con todas las combinaciones posibles
4. Sumar combinaciones relevantes para obtener probabilidades de mercado

**Mercados calculados**:

| Mercado | Calculo |
|---------|---------|
| Victoria Local | Suma de combinaciones (local > visita) |
| Empate | Suma de combinaciones (local = visita) |
| Victoria Visita | Suma de combinaciones (local < visita) |
| BTTS Si | 1 - P(local=0) - P(visita=0) + P(ambos=0) |
| Over 2.5 | Suma de combinaciones donde total > 2 |
| Under 2.5 | Suma de combinaciones donde total ≤ 2 |

**Limitacion conocida**: Poisson estandar asume independencia entre goles de los equipos. En realidad hay correlacion (ej: un equipo adelante puede jugar mas defensivo). Modelos mas sofisticados (Poisson Bivariado, Dixon-Coles) abordan esto pero agregan complejidad.

---

## Etapa 5: Deteccion de Valor

**Que hace**: Compara probabilidades calculadas contra cuotas del mercado para identificar oportunidades de valor esperado positivo.

**Por que importa**: El objetivo final no es predecir resultados—es encontrar discrepancias entre probabilidad verdadera y probabilidad implicita en las cuotas.

**Mecanismo**:
1. Convertir probabilidad a cuotas implicitas: `cuotas_implicitas = 1 / probabilidad`
2. Comparar contra cuotas del mercado
3. Si `cuotas_implicitas < cuotas_mercado` → existe valor

**Ejemplo**:
- Probabilidad calculada victoria local: 45%
- Cuotas implicitas: 1 / 0.45 = 2.22
- Cuotas del mercado: 2.50
- Como 2.22 < 2.50 → **valor detectado**

**Interpretacion**: Apostar a cuotas 2.50 cuando la probabilidad verdadera es 45% produce +12.5% de valor esperado.

---

## Diagrama de Flujo del Pipeline

```
┌─────────────────────┐
│  Stats Historicos   │
│  (xG por equipo)    │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────┐
│  1. Obtencion xG Base            │
│  Promedio ponderado (65/25/10)   │
│  de 3 temporadas                 │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  2. Ajuste Contexto Liga         │
│  Normaliza por ratio local/visita│
│  de la liga                      │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  3. Finishing Offset             │
│  xG × e^α                        │
│  (MLE + partial pooling)         │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  4. Modelo Poisson               │
│  P(k goles) para cada equipo     │
│  → Matriz marcadores → Probs     │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  5. Deteccion de Valor           │
│  cuotas_implicitas vs mercado    │
└──────────┬───────────────────────┘
           │
           ▼
┌─────────────────────┐
│  Salida:            │
│  - Probabilidades   │
│  - Flags de valor   │
└─────────────────────┘
```

---

## Razonamiento de Diseno

**¿Por que no usar un modelo end-to-end (Red Neuronal, XGBoost)?**

1. **Interpretabilidad**: Cada etapa tiene significado deportivo, facilitando debugging y comunicacion con stakeholders

2. **Modularidad**: Etapas individuales pueden mejorarse sin reentrenar todo

3. **Eficiencia de datos**: Poisson funciona bien con muestras pequenas; deep learning requeriria ordenes de magnitud mas datos

4. **Baseline solido**: Este enfoque es el estandar de la industria, validado por decadas de uso en analytics deportivo

**Extensiones posibles**:
- Incorporar features adicionales (lesiones, clima, motivacion)
- Usar correccion Dixon-Coles para correlacion de goles
- Ensemble con modelo ML para casos edge

---

## Parametros Clave

| Parametro | Valor | Razonamiento |
|-----------|-------|--------------|
| Pesos temporada | 65 / 25 / 10 | Balance recencia vs estabilidad |
| Max goles modelados | 10 | P(>10) ≈ 0 para valores xG tipicos |
| Limites offset | [-10, 10] | Rango practico (e^10 ≈ 22000x) |
| Shrinkage k | 15 | ~15 partidos para 50% confianza |
| Decimales probabilidad | 4 | Precision suficiente para cuotas |

---

*Para formulas matematicas completas, ver [Apendice Matematico](./probability-pipeline-formulas.md)*

---

*Documento preparado para portafolio publico. No contiene detalles de implementacion propietarios.*
