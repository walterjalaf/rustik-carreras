# Dashboard Ingresos por Carrera — Auditoría de KPIs y propuesta de mejora

**Proyecto:** CIAF · Conciliación de Ingresos por Carrera
**Fecha:** 2026-05-15
**Alcance:** Auditar el dashboard actual (`dashboard_carreras.html`) contra los datos disponibles en el Excel fuente, identificar gaps y proponer un set de KPIs ampliado con foco en evolución temporal.

---

## 1. Resumen ejecutivo

El dashboard hoy consume **una sola pestaña** del Excel — `ingresos x carrera` — que es un roll-up diario de **monto por carrera × medio de pago (MP / Efectivo)**. Esa pestaña está alimentada por 5 hojas con datos crudos a nivel **inscripto** (`10k`, `DVL`, `MISJ`, `DPN`, `GFM`) y una hoja de **MP** con la liquidación de Mercado Pago. Hay además una hoja `egresos` con costos.

El gap principal: el dashboard sólo muestra **plata cobrada** y deja afuera **volumen, mix, comportamiento del inscripto, descuentos, comisiones MP y rentabilidad**. Sobra dato para enriquecerlo sin pedirle nada nuevo al usuario.

---

## 2. Estado actual del dashboard

### 2.1 KPIs que muestra hoy

| # | KPI | Fórmula | Granularidad | Fuente |
|---|---|---|---|---|
| 1 | Total recaudado (hero) | Σ MP + Σ Ef de todas las carreras | Mes seleccionado | `ingresos x carrera` |
| 2 | % MP vs % Efectivo (hero) | Σ MP / Total · Σ Ef / Total | Mes | idem |
| 3 | Total por carrera (card) | MP_carrera + Ef_carrera | Mes · por carrera | idem |
| 4 | Split MP/Ef por carrera (card sub) | MP_carrera, Ef_carrera | Mes · por carrera | idem |
| 5 | % participación carrera (card) | Total_carrera / Total_general | Mes · por carrera | idem |

### 2.2 Visualizaciones que muestra hoy

| # | Visual | Dimensión | Métrica |
|---|---|---|---|
| 1 | Donut | Carrera | % del total recaudado |
| 2 | Barras agrupadas | Carrera × Medio de pago | $ MP vs $ Efectivo |
| 3 | Línea acumulada | Día × Carrera | $ acumulado diario |
| 4 | Tabla diaria | Fecha × Carrera × MP/Ef | $ por celda + subtotales |

### 2.3 Lo que NO está mostrando

- Cantidad de inscriptos (volumen) — solo monto.
- Ticket promedio.
- Mix por distancia (10K vs 21K vs 42K si aplica).
- Período de inscripción (early bird, promocional, regular).
- Procedencia geográfica de los corredores.
- Running teams.
- Uso de cupones de descuento.
- Estado de pago (approved / pending / manual / rejected).
- Comisiones MP e impacto en MP neto.
- Egresos y margen neto.
- Comparativos contra meses anteriores o ediciones previas.
- Velocidad de inscripción (run-rate) y proyección a cierre.

---

## 3. Datos disponibles en la fuente

Pestañas del Excel (`Plan_Carreras.xlsx`):

| Hoja | Tipo | Granularidad | Campos clave |
|---|---|---|---|
| `MP` | Liquidación MP | Transacción | FECHA, VALOR_COMPRA, MONTO_NETO, COMISIÓN+IVA, MEDIO_PAGO, CUOTAS, MONTO_RECIBIDO_SPLIT, IIBB |
| `egresos` | Costos | Línea de gasto | (montos negativos, ver E3=-2.794.777) |
| `ingresos` | Resumen | Mes | totales por mes |
| `10k` / `DVL` / `MISJ` / `DPN` / `GFM` | Inscriptos por carrera | Inscripción | fecha, payment_id, FirstName, LastName, **Nacion**, **Procedencia**, **running_team**, **distance**, price, **discount_code**, discounted_price, priceTax, paymentOption, payment_price, **payment_status**, payment_method_id, **periodoInscripcion**, createdAt, updatedAt, observaciones, nro_recibo, cruce |
| `ingresos x carrera` | Roll-up diario | Día × Carrera | MP, Efectivo (lo que consume hoy el dashboard) |

**Conclusión:** las hojas por carrera tienen 23 columnas a nivel inscripto. Hoy el dashboard usa **2** (las sumadas en MP/Ef). El resto está sin explotar.

---

## 4. Gaps priorizados

Ordenados por **impacto / esfuerzo de implementación**.

### Tier 1 — Sumar al hero + cards (alto impacto, bajo esfuerzo)

| Gap | Por qué importa | De dónde sale |
|---|---|---|
| **Cantidad de inscriptos** | Es el verdadero indicador de demanda; el ingreso es una consecuencia. | `COUNT(payment_id)` por hoja de carrera |
| **Ticket promedio** | Detecta si subo precio o si la promo está canibalizando | `Σ payment_price / COUNT(approved)` |
| **% inscriptos approved vs pending/manual** | Plata que parece estar pero todavía no entró | `COUNT(payment_status='approved') / total` |
| **Run-rate diario y proyección a cierre del mes** | Decisiones de marketing / cupo | Σ inscriptos por día → tendencia |

### Tier 2 — Nuevos gráficos / sección "Composición"

| Gap | Visual sugerido | Dimensión |
|---|---|---|
| **Mix por distancia** (10K / 21K / 42K) | Barras horizontales o stacked | `distance` × cantidad |
| **Mix por período de inscripción** (Promocional / Regular / Late) | Stacked bar o waterfall | `periodoInscripcion` × $ y cantidad |
| **Top procedencias** (¿de dónde vienen?) | Barras horizontales top 10 | `Procedencia` × cantidad |
| **Top running teams** | Tabla / barras | `running_team` × cantidad |
| **Uso de cupones de descuento** | KPI + barra | `discount_code` aplicado vs no, $ descontado |

### Tier 3 — Financieras (la parte que hoy no aparece)

| Gap | Por qué | Cálculo |
|---|---|---|
| **MP bruto vs MP neto** | Comisión + IVA + IIBB se comen ~5-7% según cuotas | `Σ VALOR_COMPRA - Σ MONTO_NETO` de hoja MP |
| **Total comisiones MP** | KPI directo | `Σ (COMISIÓN+IVA + IIBB)` |
| **Egresos del mes** | Hoja `egresos` | `Σ egresos` |
| **Margen neto** | El indicador que define si la carrera es negocio | `(MP_neto + Efectivo) - Egresos` |
| **Break-even (inscriptos necesarios)** | Para presupuestar próximas ediciones | `Egresos / ticket_promedio` |

### Tier 4 — Comparativos temporales (evolución, lo que pediste)

| Gap | Visual | Comparación |
|---|---|---|
| **Mes actual vs mes anterior** (header) | Δ% al lado del hero total | `(Total_M - Total_M-1) / Total_M-1` |
| **Curva acumulada vs misma fecha mes anterior** | 2 series en el line chart | Acumulado M vs acumulado M-1 hasta día N |
| **Año a año por edición** (si una carrera se repite) | Tabla edición vs edición | Misma carrera año pasado vs este |
| **Heatmap día × carrera** | Calendario coloreado por intensidad | Para ver picos (¿lunes post-Strava? ¿día de difusión?) |

---

## 5. Formato propuesto para los KPIs (ficha estándar)

Para que el dashboard tenga un set parejo y mantenible, cada KPI debería seguir esta ficha:

```
ID:            ING_TOT_MES
Nombre:        Total recaudado del mes
Descripción:   Suma de ingresos brutos (MP + Efectivo) de todas las carreras
Fórmula:       Σ (MP_i + Ef_i)  para i en días del mes
Unidad:        ARS
Granularidad:  Mes / Carrera / Día
Fuente:        ingresos x carrera (roll-up) + sheet por carrera (drill)
Comparativo:   Δ% vs mes anterior  ·  Δ% vs misma fecha mes anterior (acumulado)
Visualización: Hero (mes) + line acumulada con serie M-1 (día)
Owner:         CIAF Finanzas
Refresh:       On-demand desde Google Drive
```

### 5.1 Set propuesto completo (resumen)

| ID | Nombre | Fórmula | Visual |
|---|---|---|---|
| `ING_TOT_MES` | Total recaudado | Σ MP + Σ Ef | Hero |
| `ING_NETO_MES` | Recaudado neto post-MP | (MP_neto + Ef) | Hero secundario |
| `COM_MP_MES` | Comisiones MP | Σ (Comisión+IVA+IIBB) | Card |
| `EGR_MES` | Egresos del mes | Σ egresos | Card |
| `MARGEN_NETO` | Margen neto | Neto − Egresos | Card destacada |
| `INSC_TOT` | Inscriptos totales | COUNT(payment_id approved) | Card |
| `INSC_APPR_PCT` | % aprobados | approved / total | Card sub |
| `TICKET_PROM` | Ticket promedio | Σ price_pagado / COUNT(approved) | Card |
| `INSC_x_CARRERA` | Inscriptos por carrera | COUNT por sheet | Cards (extender las actuales) |
| `MIX_DIST` | Mix por distancia | COUNT por `distance` | Stacked bar |
| `MIX_PERIODO` | Mix por período inscripción | COUNT por `periodoInscripcion` | Stacked bar |
| `TOP_PROC` | Top procedencias | COUNT por `Procedencia` (top 10) | Barras horizontales |
| `TOP_TEAM` | Top running teams | COUNT por `running_team` (top 10) | Barras horizontales |
| `DESC_USO` | Uso de cupones | COUNT(discount_code≠null) / total | KPI + lista top códigos |
| `DESC_MONTO` | $ descontado | Σ (price − discounted_price) | Card |
| `RUN_RATE` | Velocidad inscripciones | Inscriptos / día (últimos 7d) | KPI con trend |
| `PROY_CIERRE` | Proyección a cierre | Run-rate · días restantes + actual | KPI |
| `BREAKEVEN` | Inscriptos break-even | Egresos / ticket promedio | KPI con barra de progreso |
| `Δ_VS_M-1` | Variación vs mes anterior | (M − M-1) / M-1 | Chip junto al hero |
| `HEATMAP_DC` | Calendario día × carrera | $ por celda | Heatmap |

---

## 6. Cambios concretos sugeridos al HTML

1. **Hero ampliado:** además del total bruto, agregar chip Δ vs M-1 y línea con neto + margen.
2. **Cards por carrera ampliadas:** agregar inscriptos, ticket promedio, % approved bajo el monto.
3. **Nueva sección "Composición"** (entre charts actuales y tabla): mix distancia, mix período, top procedencias, top teams.
4. **Nueva sección "Salud Financiera"**: card de egresos, comisiones MP, margen neto, break-even.
5. **Chart acumulado:** sumar serie punteada con acumulado del mes anterior (mismo día).
6. **Filtros nuevos en navbar:** distancia, período de inscripción, estado de pago.
7. **Reemplazo de la fuente de datos:** hoy lee solo `ingresos x carrera`. Para los KPIs Tier 2/3 hay que leer también las hojas de cada carrera y `MP` / `egresos`.

---

## 7. Riesgos y consideraciones

- **Calidad de datos en las hojas por carrera:** algunas filas tienen `payment_status` vacío o "manual" — necesitan regla clara para contar (ej: solo `approved` + `manual` con `nro_recibo`).
- **`Procedencia` está sin normalizar** (hay variantes "San Juan", "san juan", "SJ"). Conviene normalizar antes de agrupar.
- **`distance`** viene como texto ("10K", "21K") — castear a número o tratar como categoría.
- **`MP` vs ingresos x carrera:** la conciliación puede no cerrar 1:1 si hay pagos manuales fuera de MP. Mostrar la diferencia como KPI ("conciliación pendiente") es útil.
- **Performance:** leer 5 hojas + MP en cada carga de mes puede ser pesado. Si pesa, cachear en memoria o pre-procesar a un JSON en el sheet.

---

## 8. Próximos pasos sugeridos

1. Validar este set de KPIs con el sponsor / quien consume el dashboard.
2. Definir qué KPIs son MUST-HAVE para la próxima versión y cuáles van en backlog.
3. Cerrar la ficha de cada KPI (sección 5) con definiciones acordadas.
4. Implementar lectura de hojas adicionales en `dashboard_carreras.html`.
5. Iterar visuales con datos reales del mes vigente.
