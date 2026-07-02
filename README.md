# Interpretacion-de-Logs
Sexto challenge de the Huddle

## Descripción

Challenge de análisis de datos sobre un sistema de logging distribuido. El objetivo fue actuar como analista: dado un dataset de logs reales, detectar cuándo el sistema estuvo peor, qué servicio fue el más afectado, qué errores dominaron y qué cambió durante el incidente respecto a la normalidad del sistema.

No se trata de escribir más código, sino de sostener conclusiones con evidencia cuantitativa y reproducible.

---

## Tecnologías utilizadas

- Python 3
- Jupyter Notebook
- Pandas
- Matplotlib

---

## Estructura del proyecto

```
interpretacion_logs.ipynb   # Notebook principal, corre de arriba a abajo
server_logs.csv             # Dataset provisto, requerido para correr el notebook
README.md                   # Este archivo
```

---

## Cómo correr el notebook

1. Cloná el repositorio
2. Descargá el dataset desde el enlace provisto y colocalo en la misma carpeta que el notebook
3. Instalá las dependencias:
    Asegurate de descomentar la primera linea `pip install pandas matplotlib jupyter`
4. Ejecutá el notebook de arriba a abajo:

> El notebook es completamente reproducible. Correrlo de principio a fin produce exactamente los mismos resultados.

---

## Definiciones operativas

Las siguientes definiciones fueron aplicadas consistentemente durante todo el análisis:

| Concepto | Definición |
|----------|------------|
| **Time window** | Ventana de 5 minutos usando `timestamp_event` |
| **Bad event** | Registro con `severity == ERROR o CRITICAL` o `status_code >= 500` |
| **Bad rate** | `bad_events / total_events` por ventana |
| **Momento crítico** | Ventana con mayor `bad_rate` con al menos 20 eventos |
| **Baseline** | Todo el dataset fuera del momento crítico |

---

## Decisiones técnicas

**Columna auxiliar `is_bad`.**  
Se creó una columna booleana antes del `groupby()` para poder calcular `bad_events` dentro de `.agg()` sin necesidad de lambdas complejas que mezclen múltiples columnas.

**Criterio para endpoint más comprometido.**  
Se eligió cantidad de `status_code >= 500` porque representa fallos concretos del servidor, no simples errores de severidad. En un sistema de órdenes e inventario, un fallo de servidor implica operaciones perdidas, lo cual es más grave que lentitud.

**Conversión de timestamp.**  
Solo se convirtió `timestamp_event` a `datetime`, ya que `received_at` no se utiliza en ningún cálculo del análisis.

---

## Hallazgos principales

- **Momento crítico:** 2026-01-10 a las 11:10 UTC con un `bad_rate` de 0.58
- **Servicio más afectado:** `orders-service` con 72 bad events en la ventana crítica
- **Endpoint más comprometido:** `/orders/cancel` con la mayor cantidad de errores 5xx
- **Mensaje dominante:** `Order creation failed - inventory lock timeout` (72 ocurrencias)
- **Incidente vs Baseline:** La latencia promedio se triplicó (521ms → 1589ms) y el bad rate pasó de 14% a 58%

El análisis sugiere que el origen del incidente fue una saturación del sistema de inventario que generó timeouts en cascada hacia `orders-service`, visible desde el `api-gateway` como alta latencia antes de que los errores 5xx comenzaran a dispararse.

---

## Bonus: Rastreo de `trace_id`

Se rastreó el `trace_id` con mayor cantidad de servicios distintos involucrados dentro del momento crítico. La secuencia muestra:

1. `api-gateway` → WARN por alta latencia (1048ms), status 429
2. `orders-service` → ERROR por inventory lock timeout, status 504
3. `inventory-service` → WARN por retry de reserva, status 200

Esto confirma la hipótesis: el problema se originó en el inventario y se propagó hacia arriba en la cadena de servicios.

## Autor

**Fede Alarcón Scura** — Estudiante de Penguin Academy
Challenge desarrollado como parte del programa de formación.