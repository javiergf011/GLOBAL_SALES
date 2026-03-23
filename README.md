# 🛒 Global E-Commerce Analytics
### Análisis de ventas 2023–2025 · SQL Server · Power BI · DAX

Proyecto completo de análisis de datos de una empresa de e-commerce global con ventas en **20 países** entre **2023 y 2025**.

El proyecto cubre el ciclo completo: detección y corrección de errores de calidad en los datos, modelado en SQL Server y construcción de un dashboard interactivo en Power BI.

---

## 📸 Vista previa

<img src="CAPTURAS/CAPTURA GENERAL.png" width="800"/>
<img src="CAPTURAS/CAPTURA PAIS.png" width="800"/>
<img src="CAPTURAS/CAPTURA PRODUCTOS.png" width="800"/>

---

## 🎯 Objetivo

Demostrar un flujo completo de trabajo con datos:

1. Detectar problemas de calidad antes de analizar
2. Corregirlos de forma segura y auditable en SQL Server
3. Construir un informe profesional e interactivo en Power BI

---

## 🗂️ Estructura del proyecto

```
GLOBAL-SALES/
│
├── 📂 CAPTURAS/
│   ├── CAPTURA GENERAL.png      ← Página GENERAL del dashboard
│   ├── CAPTURA PAIS.png         ← Página PAÍS — ranking dinámico
│   └── CAPTURA PRODUCTOS.png    ← Página PRODUCTOS
│
├── 📂 CONSULTAS/
│   ├── 01_deteccion_errores.sql     ← Diagnóstico con tolerancia 1%
│   ├── 02_correccion_datos.sql      ← UPDATE + transacción + auditoría
│   ├── 03_trigger_validacion.sql    ← Trigger AFTER INSERT/UPDATE
│   └── 04_vistas_powerbi.sql        ← 8 vistas optimizadas para Power BI
│
├── 📂 DATOS/
│   ├── GLOBAL_EC.csv                ← Dataset original (2.000 registros)
│   └── GLOBAL_EC_CORREGIDO.xlsx     ← Dataset corregido con auditoría
│
├── 📄 proyecto_ecomerce.pbix        ← Informe Power BI (4 páginas)
├── 📄 ecommerce_v3_final.pptx       ← Presentación del proyecto
└── 📄 README.md
```

---

## 🔍 Hallazgo principal — Calidad de datos

Antes de realizar cualquier análisis, se auditó la columna `Total_Sales` comparándola contra el valor calculado `Quantity × Unit_Price × (1 - Discount%)`.

| Resultado | Cantidad |
|---|---|
| ✅ Registros sin error | 1.706 |
| ❌ Registros con error | **294 (14,7%)** |
| Tipo A — Total_Sales ÷10 (faltaba un cero) | 178 |
| Tipo B — Total_Sales ×10 (sobraba un cero) | 116 |

> **Patrón identificado:** todos los errores son desplazamientos de exactamente un decimal. El patrón es 100% consistente — no existe ningún otro tipo de discrepancia.

---

## 🛠️ Solución SQL Server

### 1. Diagnóstico
```sql
SELECT
    Order_ID,
    Total_Sales,
    ROUND(Quantity * Unit_Price * (1.0 - Discount_Percent / 100.0), 2) AS Esperado,
    ABS(Total_Sales - Quantity * Unit_Price * (1.0 - Discount_Percent / 100.0))
    / NULLIF(CAST(Total_Sales AS FLOAT), 0) * 100                      AS Diferencia_Pct
FROM orders
WHERE
    ABS(Total_Sales - Quantity * Unit_Price * (1.0 - Discount_Percent / 100.0))
    / NULLIF(CAST(Total_Sales AS FLOAT), 0) * 100 > 1;
```

### 2. Corrección con transacción
```sql
BEGIN TRANSACTION;

    UPDATE orders
    SET Total_Sales = ROUND(Quantity * Unit_Price * (1.0 - Discount_Percent / 100.0), 2)
    WHERE
        ABS(Total_Sales - Quantity * Unit_Price * (1.0 - Discount_Percent / 100.0))
        / NULLIF(CAST(Total_Sales AS FLOAT), 0) * 100 > 1;

    PRINT 'Filas corregidas: ' + CAST(@@ROWCOUNT AS VARCHAR);

COMMIT;
```

### 3. Trigger de prevención futura

El trigger `trg_validar_total_sales` se ejecuta automáticamente en cada `INSERT` o `UPDATE`:

| Desviación | Acción |
|---|---|
| `< 1%` | ✅ Aceptar — diferencia por redondeo |
| `1% – 1000%` | 🔧 Corregir automáticamente |
| `> 1000%` | 🚫 ROLLBACK + mensaje de error |

---

## 📊 Dashboard Power BI

### Páginas del informe

| Página | Contenido |
|---|---|
| **INFORMACIÓN GENERAL** | KPIs globales, categorías, hallazgos clave, métodos de pago |
| **GENERAL** | Tendencia mensual de ventas vs año anterior, top países por beneficio |
| **PAÍS** | Ranking dinámico Top N, variación YoY, países destacados |
| **PRODUCTOS** | Unidades por categoría, tabla de productos con rentabilidad |

### Medidas DAX principales

```dax
-- Ranking dinámico que respeta los filtros activos del informe
Ranking Pais Ventas =
IF(
    ISINSCOPE(ecomerce[Country]),
    RANKX(ALLSELECTED(ecomerce[Country]), [Ventas Totales], , DESC, DENSE),
    BLANK()
)

-- Variación Year-over-Year
DIF Ventas =
DIVIDE([Ventas Totales] - [Ventas LY], [Ventas LY], 0)

-- Top N dinámico controlado por segmentador
Ventas Top N =
VAR N = SELECTEDVALUE(TopN[Valor], 10)
RETURN IF([Ranking Pais Ventas] <= N, [Ventas Totales], BLANK())
```

### Vistas SQL creadas para Power BI

| Vista | Descripción |
|---|---|
| `vw_ventas_por_mes` | Evolución mensual de ventas y margen |
| `vw_ventas_por_producto` | Ranking de productos con clasificación |
| `vw_ventas_por_pais` | KPIs geográficos por país y región |
| `vw_ventas_por_segmento` | Comparativa Corporate / Consumer / Home Office |
| `vw_impacto_descuentos` | Efecto del descuento sobre el margen |
| `vw_metodos_pago` | Ticket medio por método de pago |
| `vw_ranking_producto_pais` | Top productos dentro de cada país |
| `vw_crecimiento_yoy` | Crecimiento Year-over-Year por categoría |

---

## 📈 Hallazgos del análisis

- 📈 **Crecimiento YoY del 53,84%** — $855M vs $556M en el periodo anterior
- 🥇 **United States** lidera ventas con $97,9M — el 29% del total global
- 🚀 **France** registra el mayor crecimiento: ▲ 361,38% (de $11,9M a $55,2M)
- 👥 **Consumer** representa el 50% de todos los pedidos (1.006 de 2.000)
- 💳 **Credit Card** es el método de pago dominante con el 39,8% de transacciones
- 🚚 **Middle East & Africa** tiene el mayor coste medio de envío: $1.487 por pedido

---

## 🛠️ Stack tecnológico

![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=flat&logo=microsoftsqlserver&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Excel](https://img.shields.io/badge/Excel-217346?style=flat&logo=microsoftexcel&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)

---

## 👤 Autor

**Javier García Fernández**
[LinkedIn](https://www.linkedin.com/in/javier-garcia-a225313b6) · [GitHub](https://github.com/javiergf011)

---
