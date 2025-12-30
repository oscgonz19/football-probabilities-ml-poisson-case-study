# Pipeline de Probabilidades: Formulas Matematicas

> Referencia Compacta para Estadisticos y Quants

---

## 1. Promedio xG Ponderado

```
xG_ponderado = Σ(wᵢ × xGᵢ) / Σwᵢ

donde:
  w = [65, 25, 10]  (temporada actual, -1, -2)

  w₀_ajustado = w₀ × ratio_completitud
  ratio_completitud = partidos_jugados / partidos_esperados
```

Si una temporada carece de datos validos (`xG = 0` o `null`), su peso se redistribuye proporcionalmente a temporadas con datos.

---

## 2. Ajuste por Contexto de Liga

```
ratio = xG_promedio_local_liga / xG_promedio_visita_liga

xG_local = (xG_favor_local × ratio + xG_contra_visita_oponente × (1/ratio)) / 2
xG_visita = (xG_favor_visita × (1/ratio) + xG_contra_local_oponente × ratio) / 2
```

Esto balancea expectativa ofensiva con debilidad defensiva del oponente, normalizado por promedios de liga.

---

## 3. Finishing Offset

### 3.1 Log-Likelihood (Poisson)

```
ℓ(α) = Σᵢ [-λᵢ + gᵢ × ln(λᵢ) - ln(gᵢ!)]

donde:
  λᵢ = xGᵢ × e^α
  gᵢ = goles reales en partido i
  α  = finishing offset (parametro a optimizar)
```

### 3.2 Optimizacion

```
α_raw = argmax ℓ(α)
        α ∈ [-10, 10]

Metodo: L-BFGS-B (quasi-Newton con limites)
```

### 3.3 Partial Pooling (Regularizacion)

```
α_final = α_raw × (n / (n + k))

donde:
  n = numero de partidos del equipo
  k = 15 (hiperparametro de shrinkage)
```

**Comportamiento**:
- Cuando `n → 0`: `α_final → 0` (prior neutral)
- Cuando `n → ∞`: `α_final → α_raw` (confia en datos empiricos)

### 3.4 Aplicacion

```
xG_final = xG_ajustado × e^(α_final)
```

---

## 4. Distribucion Poisson

### 4.1 Funcion de Masa de Probabilidad

```
P(X = k | λ) = (e^(-λ) × λ^k) / k!

donde:
  X = numero de goles
  λ = xG_final (goles esperados)
  k ∈ {0, 1, 2, ..., 10}
```

### 4.2 Distribuciones por Equipo

```
P_local(k) = P(X = k | λ_local)
P_visita(k) = P(X = k | λ_visita)
```

### 4.3 Matriz de Marcadores

```
M[i,j] = P_local(i) × P_visita(j)

     Goles Visita
      0      1      2      3    ...
Local:
  0   M[0,0]  M[0,1]  M[0,2]  M[0,3]
  1   M[1,0]  M[1,1]  M[1,2]  M[1,3]
  2   M[2,0]  M[2,1]  M[2,2]  M[2,3]
  ...

Nota: Asume independencia entre goles local y visita
```

---

## 5. Probabilidades de Mercado

### 5.1 Resultado del Partido (1X2)

```
P(Victoria Local) = Σ M[i,j]  ∀ i > j
P(Empate)         = Σ M[i,i]  ∀ i
P(Victoria Visita)= Σ M[i,j]  ∀ i < j

Verificacion: P(L) + P(E) + P(V) = 1
```

### 5.2 Ambos Equipos Anotan (BTTS)

```
P(BTTS Si) = 1 - P_local(0) - P_visita(0) + M[0,0]
P(BTTS No) = 1 - P(BTTS Si)

Forma alternativa:
P(BTTS Si) = Σ M[i,j]  ∀ i > 0, j > 0

Verificacion: P(Si) + P(No) = 1
```

### 5.3 Over/Under Total de Goles

```
P(Over N)  = Σ M[i,j]  ∀ (i + j) > N
P(Under N) = Σ M[i,j]  ∀ (i + j) ≤ N

Ejemplo - Over 2.5:
P(Over 2.5) = 1 - [M[0,0] + M[1,0] + M[0,1] + M[2,0] + M[1,1] + M[0,2]]
            = 1 - Σ M[i,j] donde i + j ≤ 2

Verificacion: P(Over N) + P(Under N) = 1
```

---

## 6. Deteccion de Valor

### 6.1 Cuotas Implicitas

```
cuotas_implicitas = 1 / P_calculada
```

### 6.2 Condicion de Valor

```
existe_valor = (cuotas_implicitas < cuotas_mercado)

Equivalente:
existe_valor = (P_calculada > 1 / cuotas_mercado)
```

### 6.3 Valor Esperado

```
EV = (P_calculada × cuotas_mercado) - 1

Ejemplo:
P = 0.45, cuotas = 2.50
EV = (0.45 × 2.50) - 1 = 0.125 = +12.5%
```

### 6.4 Edge (Ventaja)

```
edge = P_calculada - P_implicita
     = P_calculada - (1 / cuotas_mercado)

Ejemplo:
edge = 0.45 - (1/2.50) = 0.45 - 0.40 = 0.05 = 5%
```

---

## 7. Pseudocodigo Completo del Pipeline

```python
def calcular_probabilidades_partido(partido):
    # 1. Obtener xG ponderado (3 temporadas)
    xg_local = promedio_ponderado(
        equipo=partido.equipo_local,
        temporadas=ultimas_3_temporadas,
        pesos=[65, 25, 10],
        ajustar_por_completitud=True
    )
    xg_visita = promedio_ponderado(partido.equipo_visita, ...)

    # 2. Ajuste por contexto de liga
    ratio = partido.liga.xg_promedio_local / partido.liga.xg_promedio_visita
    xg_local = (xg_local * ratio + contra_oponente * (1/ratio)) / 2
    xg_visita = (xg_visita * (1/ratio) + contra_oponente * ratio) / 2

    # 3. Finishing offset
    offset_local = obtener_finishing_offset(partido.equipo_local)  # MLE + shrinkage
    offset_visita = obtener_finishing_offset(partido.equipo_visita)
    xg_local *= exp(offset_local)
    xg_visita *= exp(offset_visita)

    # 4. Distribuciones Poisson
    dist_local = [poisson_pmf(k, xg_local) for k in range(11)]
    dist_visita = [poisson_pmf(k, xg_visita) for k in range(11)]

    # 5. Matriz de marcadores
    matriz = [[dist_local[i] * dist_visita[j]
               for j in range(11)]
               for i in range(11)]

    # 6. Probabilidades de mercado
    prob_local = sum(matriz[i][j] for i in range(11) for j in range(i))
    prob_empate = sum(matriz[i][i] for i in range(11))
    prob_visita = sum(matriz[i][j] for i in range(11) for j in range(i+1, 11))

    prob_btts_si = sum(matriz[i][j] for i in range(1,11) for j in range(1,11))
    prob_btts_no = 1 - prob_btts_si

    prob_over_25 = sum(matriz[i][j] for i in range(11) for j in range(11) if i+j > 2)
    prob_under_25 = 1 - prob_over_25

    # 7. Deteccion de valor
    valor_local = (1/prob_local) < partido.cuotas_local
    valor_empate = (1/prob_empate) < partido.cuotas_empate
    valor_visita = (1/prob_visita) < partido.cuotas_visita

    return {
        'probabilidades': {...},
        'flags_valor': {...}
    }
```

---

## 8. Resumen de Parametros

| Parametro | Valor | Rol Matematico |
|-----------|-------|----------------|
| Pesos temporada | [65, 25, 10] | Coeficientes de promedio ponderado |
| Max k (goles) | 10 | Truncamiento de suma Poisson infinita |
| Limites offset | [-10, 10] | Dominio de optimizacion restringido |
| Shrinkage k | 15 | Fuerza de regularizacion en partial pooling |
| Tolerancia ε | 0.05 | Desviacion aceptable de suma = 1.0 |

---

## 9. Limitaciones Conocidas

1. **Supuesto de independencia**: Poisson estandar asume que goles local y visita son independientes. Correlacion existe en la practica.

2. **xG estatico**: No considera dinamicas dentro del partido (tarjetas rojas, lesiones, cambios tacticos).

3. **Sin inflacion de empate**: Poisson subestima ligeramente empates. Correccion Dixon-Coles aborda esto.

4. **Estabilidad de finishing offset**: Muestras pequenas pueden producir offsets volatiles a pesar de regularizacion.

---

*Documento preparado para portafolio publico. Contiene solo formulas estadisticas estandar.*
