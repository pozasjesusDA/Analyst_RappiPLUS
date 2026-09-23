# RappiPlus: análisis integral de negocio

Proyecto de análisis de datos desarrollado para evaluar el desempeño comercial de **RappiPlus**, identificar oportunidades de mejora en el proceso de compra, analizar la retención de usuarios y validar el impacto de una modificación en la interfaz del checkout.

El proyecto integra **Python, SQL, estadística y Power BI** para transformar datos transaccionales en indicadores y visualizaciones útiles para la toma de decisiones.

## Objetivos

- Limpiar y validar los datos de ventas, catálogo y marketing.
- Medir ingresos, costos, inversión en marketing y rentabilidad.
- Analizar el comportamiento de compra por producto.
- Construir un funnel de conversión e identificar sus principales pérdidas.
- Evaluar la retención semanal mediante cohortes mensuales.
- Determinar mediante una prueba estadística si una nueva interfaz de checkout modifica la conversión.
- Comunicar los resultados en un dashboard interactivo de Power BI.

## Fuentes de datos

| Dataset | Descripción |
|---|---|
| `orders_clean.csv` | Pedidos, usuarios, fechas, productos, cantidades, precios y montos de venta. |
| `catalog_clean.csv` | Catálogo de productos, categorías, proveedores y costos unitarios. |
| `marketing_clean.csv` | Inversión de marketing por fecha, país, campaña y canal. |
| `events` | Eventos de navegación utilizados para construir el funnel de conversión. |
| `users` | Usuarios y fechas de registro para definir cohortes. |
| `user_activity` | Actividad semanal posterior al registro. |
| `experiment_checkout_ui.csv` | Resultados del experimento de interfaz para los grupos control y tratamiento. |

Las tablas `events`, `users` y `user_activity` fueron consultadas desde PostgreSQL.

## Herramientas utilizadas

- **Python:** pandas y Jupyter Notebook.
- **SQL / PostgreSQL:** consultas, agregaciones, CTE y funciones de ventana.
- **Estadística:** prueba Z para dos proporciones con `statsmodels`.
- **Power BI:** modelado, DAX, inteligencia de tiempo, visualizaciones y drill-through.
- **GitHub:** documentación y control de versiones.

## Proceso de análisis

### 1. Preparación y validación de datos

Se revisaron tipos de datos, valores ausentes, duplicados y consistencia de las categorías. Después de la limpieza se obtuvieron:

| Dataset | Dimensiones finales | Ausentes | Duplicados |
|---|---:|---:|---:|
| Orders | 24,906 × 12 | 0 | 0 |
| Catalog | 7 × 4 | 0 | 0 |
| Marketing | 1,620 × 5 | 0 | 0 |

### 2. KPIs de rentabilidad y ventas

Se combinaron los pedidos con el catálogo para incorporar el costo unitario de cada producto.

| Indicador | Resultado |
|---|---:|
| Revenue total | $9,610,018.94 |
| Costo total | $3,828,869.01 |
| Gasto de marketing | $2,871,843.53 |
| Profit total | $2,909,306.40 |
| Ticket promedio | $385.85 |
| Productos promedio por orden | 1.50 |

La rentabilidad general se calculó como:

```text
Profit = Revenue - Costo de productos - Gasto de marketing
```

### 3. Funnel de conversión

El flujo analizado fue:

```text
first_visit → select_item → add_to_cart → begin_checkout → add_payment_info → purchase
```

La tasa de conversión acumulada desde la primera visita hasta la compra fue de **80.04 %**. La mayor pérdida ocurrió entre `begin_checkout` y `add_payment_info`, con **958 usuarios perdidos**.

> Nota: el conteo independiente de usuarios por evento puede generar incrementos entre etapas si algunos usuarios activan un evento posterior sin tener registrado el anterior. Para un funnel estrictamente secuencial sería necesario validar también el orden temporal por usuario.

### 4. Retención por cohortes

Los usuarios fueron agrupados según el mes de registro y se calculó su actividad durante las semanas 1, 2 y 3.

Los porcentajes observados se mantuvieron aproximadamente entre **40 % y 44 %**, sin una caída pronunciada entre las tres primeras semanas. Este resultado permite comparar la permanencia de las distintas cohortes y detectar cambios en el comportamiento de los usuarios.

### 5. Experimento sobre la interfaz del checkout

Se evaluó si la interfaz modificada afectaba la tasa de conversión.

- **H₀:** no existe diferencia en la tasa de conversión entre control y tratamiento.
- **H₁:** existe una diferencia en la tasa de conversión entre ambos grupos.
- **Prueba aplicada:** prueba Z de dos proporciones.
- **Nivel de significancia:** α = 0.05.

| Grupo | Tasa de conversión |
|---|---:|
| Control | 15.69 % |
| Tratamiento | 16.29 % |

Resultados de la prueba:

- Estadístico Z: **-0.8133**.
- Valor p: **0.4161**.
- Diferencia observada: **0.60 puntos porcentuales** a favor del tratamiento.

Como el valor p es mayor que 0.05, **no se rechaza H₀**. La diferencia observada no constituye evidencia estadística suficiente para afirmar que la nueva interfaz mejoró la conversión.

## Dashboard de Power BI

### Página 1: Overview Ejecutivo

Incluye:

- Tarjetas de revenue, profit, gasto de marketing, ticket promedio y productos por orden.
- Evolución mensual de revenue y profit.
- Revenue acumulado YTD.
- Revenue y profit antes de marketing por producto.

### Página 2: Detalle de Producto y Órdenes

Incluye:

- Segmentadores por fecha, categoría y producto.
- Cantidad vendida por producto.
- Tarjetas de revenue, costo y profit antes de marketing.
- Tabla detallada por pedido.
- Formato condicional del profit: rojo para valores negativos y verde para positivos.
- Drill-through desde el Overview hacia el detalle del producto seleccionado.

### Modelo de datos

- `catalog_clean[nombre_producto]` 1 → * `orders_clean[nombre_producto]`.
- `Dim_Fecha[Date]` 1 → * `orders_clean[fecha_hora_pedido]`.
- `Dim_Fecha[Date]` 1 → * `marketing_clean[fecha]`.

## Principales medidas DAX

```DAX
Revenue Total =
SUM(orders_clean[monto_total])

Costo Total =
SUMX(
    orders_clean,
    orders_clean[cantidad] * RELATED(catalog_clean[costo_unitario])
)

Gasto Marketing =
SUM(marketing_clean[gasto])

Profit Total =
[Revenue Total] - [Costo Total] - [Gasto Marketing]

Total Pedidos =
DISTINCTCOUNT(orders_clean[id_pedido])

Cantidad Vendida =
SUM(orders_clean[cantidad])

Ticket Promedio =
DIVIDE([Revenue Total], [Total Pedidos], 0)

Productos Promedio por Orden =
DIVIDE([Cantidad Vendida], [Total Pedidos], 0)

Revenue YTD =
TOTALYTD([Revenue Total], Dim_Fecha[Date])

Profit antes de marketing =
[Revenue Total] - [Costo Total]
```

## Principales conclusiones

- El negocio obtuvo un profit total positivo de aproximadamente **$2.91 millones** después de costos y marketing.
- La mayor oportunidad del funnel está antes de ingresar la información de pago.
- La conversión final del funnel fue de **80.04 %**.
- La retención semanal se mantuvo relativamente estable durante las primeras tres semanas.
- Aunque el tratamiento del experimento presentó una conversión ligeramente superior, la diferencia no fue estadísticamente significativa.
- El análisis por producto permite identificar productos con profit negativo y apoyar decisiones sobre precios, costos y continuidad del catálogo.

## Recomendaciones

- Revisar fricciones, errores y requisitos en la transición hacia `add_payment_info`.
- Analizar los productos con profit negativo y evaluar ajustes de precio o negociación de costos.
- No implementar globalmente la nueva interfaz basándose únicamente en este experimento; considerar una nueva prueba con mayor tamaño de muestra o una mejora de mayor magnitud.
- Dar seguimiento a la retención durante períodos más largos y segmentarla por país, dispositivo y tipo de plan.
- Definir un método de atribución de marketing antes de calcular profit neto por producto o canal.

## Estructura sugerida del repositorio

```text
rappiplus-analisis/
├── README.md
├── data/
│   ├── orders_clean.csv
│   ├── catalog_clean.csv
│   ├── marketing_clean.csv
│   └── experiment_checkout_ui.csv
├── notebooks/
│   └── analisis_rappiplus.ipynb
├── sql/
│   ├── funnel_conversion.sql
│   └── cohort_retention.sql
├── dashboard/
│   └── rappiplus_dashboard.pbix
└── images/
    ├── overview_ejecutivo.png
    └── detalle_producto.png
```

## Cómo utilizar el proyecto

1. Clonar o descargar el repositorio.
2. Abrir el notebook para consultar la limpieza, los KPIs y la prueba estadística.
3. Revisar las consultas SQL del funnel y la retención por cohortes.
4. Abrir el archivo `.pbix` con Power BI Desktop.
5. Si Power BI no encuentra los CSV, modificar la ruta de origen desde **Transformar datos → Configuración de origen de datos** y actualizar el modelo.

## Autor

**Jesús Pozas Rivera**  
Proyecto académico de análisis de datos — TripleTen.
