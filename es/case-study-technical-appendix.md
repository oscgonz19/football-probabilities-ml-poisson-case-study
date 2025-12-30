# Sistema de Prediccion de Probabilidades de Futbol: Anexo Tecnico

> Caso de Estudio - Documentacion Tecnica Detallada

---

## 1. Vision General del Sistema

El sistema de prediccion calcula probabilidades de partidos usando un enfoque hibrido que combina predicciones ML externas con metodos estadisticos tradicionales. La arquitectura prioriza disponibilidad a traves de fallback automatico, valida todas las salidas contra invariantes estrictos, y maneja respuestas parciales o malformadas gracefully.

### Principios de Diseno Core

- **Fail-safe por defecto**: Cualquier falla de servicio externo dispara fallback a calculos internos
- **Validar todo**: Todas las probabilidades se verifican contra invariantes matematicos antes de persistir
- **Loggear con contexto**: Cada operacion incluye identificadores para debugging sin exponer datos sensibles
- **Testear en fronteras**: Tests unitarios cubren logica; tests de regresion validan contratos externos

---

## 2. Arquitectura del Flujo de Prediccion

### 2.1 Flujo de Alto Nivel

```
Entrada: Match
    │
    ▼
┌─────────────────────┐
│  Filtro Climatico   │ → Marca condiciones de clima de alto riesgo
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Asignacion xG      │ → Asigna goles esperados pre-partido
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│  Prediccion ML      │────▶│  Validador de       │
│  (si habilitado)    │     │  Respuesta          │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │ falla/timeout             │ invalido
          ▼                           ▼
┌─────────────────────┐     ┌─────────────────────┐
│  Calculo Tradicional│◀────│  Llenado Parcial    │
│  (basado en Poisson)│     │  (respuesta hibrida)│
└─────────┬───────────┘     └─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  Chequeo Invariantes│ → Valida sumas, rangos, consistencia
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Deteccion de Valor │ → Compara vs cuotas del mercado
└─────────┬───────────┘
          │
          ▼
    Persistir Resultados
```

### 2.2 Estrategia de Fallback

El sistema implementa una estrategia de fallback de tres niveles:

| Nivel | Condicion | Accion |
|-------|-----------|--------|
| 1 | Servicio ML retorna respuesta valida completa | Usa predicciones ML directamente |
| 2 | Servicio ML retorna respuesta parcial | Usa ML para mercados disponibles, llena faltantes con Poisson |
| 3 | Servicio ML falla/timeout/no disponible | Usa calculos Poisson para todos los mercados |

**El fallback es automatico y no requiere intervencion de operador.**

---

## 3. Invariantes de Validacion

Todas las salidas de probabilidad deben satisfacer estos invariantes antes de persistir:

### 3.1 Invariantes de Rango

```
Para todos los valores de probabilidad P:
  0.0 ≤ P ≤ 1.0
```

### 3.2 Invariantes de Suma

```
Mercado 1X2:
  |P(local) + P(empate) + P(visita) - 1.0| < ε
  donde ε = 0.05 (tolerancia para redondeo)

Mercado BTTS:
  |P(si) + P(no) - 1.0| < ε

Mercado Over/Under (para cada umbral):
  |P(over) + P(under) - 1.0| < ε
```

### 3.3 Chequeos de Sanidad xG

```
Para valores de goles esperados:
  0.0 ≤ xG < 10.0  (limite superior: ningun equipo espera realistamente 10+ goles)
```

### 3.4 Pseudocodigo de Validacion

```python
def validar_probabilidades(datos_partido):
    errores = []

    # Chequeos de rango
    campos_prob = ['prob_local', 'prob_empate', 'prob_visita',
                   'prob_btts_si', 'prob_btts_no',
                   'prob_over_25', 'prob_under_25']

    for campo in campos_prob:
        valor = datos_partido.get(campo)
        if valor is not None:
            if not (0.0 <= valor <= 1.0):
                errores.append(f"{campo} fuera de rango: {valor}")

    # Chequeos de suma
    TOLERANCIA = 0.05

    suma_1x2 = sum([datos_partido.get('prob_local', 0),
                    datos_partido.get('prob_empate', 0),
                    datos_partido.get('prob_visita', 0)])
    if abs(suma_1x2 - 1.0) > TOLERANCIA:
        errores.append(f"Suma 1X2 invalida: {suma_1x2}")

    suma_btts = sum([datos_partido.get('prob_btts_si', 0),
                     datos_partido.get('prob_btts_no', 0)])
    if abs(suma_btts - 1.0) > TOLERANCIA:
        errores.append(f"Suma BTTS invalida: {suma_btts}")

    # Sanidad xG
    for campo_xg in ['xg_local', 'xg_visita']:
        xg = datos_partido.get(campo_xg)
        if xg is not None and not (0.0 <= xg < 10.0):
            errores.append(f"{campo_xg} no razonable: {xg}")

    return errores
```

---

## 4. Implementacion de Fallback

### 4.1 Pseudocodigo de Prediccion con Fallback

```python
def predecir_con_fallback(partido):
    # Paso 1: Asignar xG pre-partido (siempre requerido)
    asignar_xg_prepartido(partido)

    # Paso 2: Verificar si predicciones ML estan habilitadas
    if not servicio_ml_habilitado():
        return calcular_tradicional(partido)

    # Paso 3: Intentar prediccion ML
    try:
        respuesta = cliente_ml.predecir(
            partido_id=partido.id,
            equipo_local=partido.equipo_local,
            equipo_visita=partido.equipo_visita,
            fecha_partido=partido.fecha,
            liga=partido.liga
        )

        # Paso 4: Validar estructura de respuesta
        if not es_respuesta_valida(respuesta):
            log_warning(f"Respuesta ML invalida para partido {partido.id}")
            return calcular_tradicional(partido)

        # Paso 5: Verificar respuesta parcial
        resultado = extraer_predicciones_ml(respuesta)
        mercados_faltantes = encontrar_mercados_faltantes(resultado)

        if mercados_faltantes:
            log_warning(f"Respuesta ML parcial, llenando: {mercados_faltantes}")
            tradicional = calcular_tradicional(partido)
            resultado = merge_predicciones(resultado, tradicional, mercados_faltantes)

        return resultado

    except TimeoutError:
        log_warning(f"Timeout ML para partido {partido.id}, usando fallback")
        return calcular_tradicional(partido)

    except ErrorServicio as e:
        log_warning(f"Error ML para partido {partido.id}: {e}, usando fallback")
        return calcular_tradicional(partido)

    except Exception as e:
        log_error(f"Error inesperado para partido {partido.id}: {e}")
        return calcular_tradicional(partido)


def es_respuesta_valida(respuesta):
    """Verifica que respuesta tenga estructura minima requerida"""
    if not isinstance(respuesta, dict):
        return False

    probs = respuesta.get('probs')
    if not probs:
        return False

    # Debe tener al menos probabilidades 1X2
    requeridos = ['local', 'empate', 'visita']
    for key in requeridos:
        if key not in probs or not isinstance(probs[key], (int, float)):
            return False

    return True


def encontrar_mercados_faltantes(resultado):
    """Identifica que mercados necesitan calculo fallback"""
    faltantes = []

    if resultado.get('prob_btts_si') is None:
        faltantes.append('btts')

    if resultado.get('prob_over_25') is None:
        faltantes.append('over_under_25')

    if resultado.get('prob_over_15') is None:
        faltantes.append('over_under_15')

    return faltantes
```

---

## 5. Manejo de Respuestas Parciales

El sistema tolera respuestas ML incompletas por diseno:

| Respuesta ML Contiene | Accion |
|----------------------|--------|
| Solo 1X2 | Acepta 1X2, calcula BTTS y O/U tradicionalmente |
| 1X2 + BTTS | Acepta ambos, calcula O/U tradicionalmente |
| 1X2 + O/U 2.5 solamente | Acepta ambos, calcula BTTS y O/U 1.5 tradicionalmente |
| Vacia/malformada | Registra warning, usa calculo tradicional completo |

**Razon**: Esto permite que el servicio ML evolucione independientemente. Nuevos mercados pueden agregarse a la respuesta ML sin requerir cambios de codigo en el consumidor.

---

## 6. Enfoque de Testing de Contrato

### 6.1 Piramide de Tests

```
         ┌─────────────┐
         │  Regresion  │  ← Llamadas HTTP reales a servicio vivo
         │    Tests    │     (corren bajo demanda, no en CI)
         └──────┬──────┘
                │
         ┌──────┴──────┐
         │ Integracion │  ← HTTP mockeado con fixtures realistas
         │    Tests    │     (corren en CI)
         └──────┬──────┘
                │
         ┌──────┴──────┐
         │  Unitarios  │  ← Tests de logica pura
         │    Tests    │     (corren en CI)
         └─────────────┘
```

### 6.2 Contrato de Test de Regresion

Tests de regresion validan contra una instancia viva del servicio ML:

```python
# Contrato: probabilidades 1X2
assert respuesta.prob_local is not None
assert respuesta.prob_empate is not None
assert respuesta.prob_visita is not None
assert 0.0 <= respuesta.prob_local <= 1.0
assert 0.0 <= respuesta.prob_empate <= 1.0
assert 0.0 <= respuesta.prob_visita <= 1.0
assert abs(respuesta.prob_local + respuesta.prob_empate + respuesta.prob_visita - 1.0) < 0.05

# Contrato: probabilidades BTTS
assert respuesta.prob_btts_si is not None
assert respuesta.prob_btts_no is not None
assert abs(respuesta.prob_btts_si + respuesta.prob_btts_no - 1.0) < 0.05

# Contrato: valores xG
assert respuesta.xg_local >= 0.0
assert respuesta.xg_local < 10.0
assert respuesta.xg_visita >= 0.0
assert respuesta.xg_visita < 10.0
```

### 6.3 Aislamiento de Tests

- **Tests de regresion** corren solo con flag explicito (`RUN_REGRESSION=1`)
- **Pipeline CI** corre solo tests unitarios e integracion (sin dependencias externas)
- **Fallas de regresion** no bloquean deploys pero disparan alertas

---

## 7. Riesgos Operativos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigacion |
|--------|--------------|---------|------------|
| Caida servicio ML | Media | Bajo | Fallback automatico a Poisson; sin intervencion manual necesaria |
| Servicio ML retorna datos invalidos | Baja | Media | Validacion estricta antes de persistir; fallback en falla |
| Pico de latencia servicio ML | Media | Bajo | Timeout 15 segundos; fallback inmediato en timeout |
| Particion de red | Baja | Bajo | Retry con backoff exponencial (2 intentos); luego fallback |
| Cambios de contrato ML | Media | Media | Tolerancia a respuesta parcial; tests de regresion capturan cambios breaking |
| API clima no disponible | Baja | Bajo | Partidos procesados sin clima; marcados para actualizacion posterior |
| Problemas conexion DB | Baja | Alto | Connection pooling estandar Rails; logica de retry |
| Rate limit excedido en API datos | Media | Media | Rate limiting adaptativo; backoff en respuestas 429 |

### 7.1 Recomendaciones de Monitoreo

- **Alertar en**: Tasa de fallback > 20% en 1 hora (indica problemas servicio ML)
- **Alertar en**: Tasa de falla de validacion > 1% (indica problemas calidad de datos)
- **Dashboard**: Mostrar distribucion de fuente de prediccion (ML vs tradicional)
- **Analisis de logs**: Trackear percentiles de latencia por fuente de prediccion

---

## 8. Caracteristicas de Performance

| Operacion | Latencia Tipica | Notas |
|-----------|-----------------|-------|
| Prediccion ML (exito) | 200-500ms | Limitado por red |
| Prediccion ML (timeout) | 15,000ms | Techo configurado |
| Calculo tradicional | 5-20ms | Limitado por CPU, paralelizable |
| Procesamiento completo partido | 50-100ms | Incluyendo escrituras DB |
| Procesamiento batch (100 partidos) | 10-30s | Con 4 workers paralelos |

---

## 9. Stack Tecnologico

| Capa | Tecnologia | Proposito |
|------|------------|-----------|
| Aplicacion | Ruby on Rails | Framework web, orquestacion de jobs |
| Base de datos | PostgreSQL | Almacenamiento persistente con constraints |
| Cliente HTTP | Faraday | Llamadas API externas con middleware |
| Testing | RSpec, WebMock | Testing unitario/integracion |
| Optimizacion | L-BFGS-B | Calibracion de finishing offset |
| Numerico | Numo::NArray | Operaciones de arrays para estadisticas |

---

*Documento preparado para portafolio publico. No contiene credenciales, URLs internas, ni detalles de implementacion propietarios.*
