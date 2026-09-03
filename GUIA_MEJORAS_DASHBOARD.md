# Guía de mejoras — Dashboard "Ppto Gastos"

Basada en el análisis real del archivo `Power BI Ppto Gastos.pbix` (formato PBIR).
Modelo actual: 3 tablas — `Gastos_2026` (hechos), `Grupo_Cta_Ctble` (dimensión),
`Responsable_Cod_CC` (dimensión). **No hay tabla/columna de presupuesto aprobado**,
así que las medidas de "control" usan `Proyección Anualizada de Gasto` como
referencia temporal, prorrateada por el avance del año.

> Nota: no hay tabla de calendario (`Date table`). Las medidas de abajo asumen
> que `Gastos_2026[MES]` es numérico (1-12). Si es texto, avísame para ajustarlas.

---

## 1. Medidas DAX nuevas (control con lo que ya tienes)

Créalas en la tabla `Gastos_2026` (Modelado → Nueva medida):

```dax
Meses Transcurridos =
CALCULATE ( MAX ( Gastos_2026[MES] ), ALLSELECTED ( Gastos_2026 ) )

Meses Restantes = 12 - [Meses Transcurridos]

% Avance del Año = DIVIDE ( [Meses Transcurridos], 12 )

Gasto Acumulado YTD =
VAR MesActual = MAX ( Gastos_2026[MES] )
RETURN
    CALCULATE (
        [Gasto Real Total],
        FILTER ( ALLSELECTED ( Gastos_2026[MES] ), Gastos_2026[MES] <= MesActual )
    )

Gasto Esperado a la Fecha =
[Proyección Anualizada de Gasto] * [% Avance del Año]

Variación vs Proyección Prorrateada % =
DIVIDE (
    [Gasto Acumulado YTD] - [Gasto Esperado a la Fecha],
    [Gasto Esperado a la Fecha]
)

Semáforo de Control =
VAR V = [Variación vs Proyección Prorrateada %]
RETURN
    SWITCH (
        TRUE (),
        V > 0.15, "🔴 Sobre-ejecutado",
        V > 0.05, "🟡 Alerta",
        V < -0.15, "🔵 Sub-ejecutado",
        "🟢 En control"
    )

Gasto Mes Anterior =
VAR MesActual = MAX ( Gastos_2026[MES] )
RETURN
    CALCULATE (
        [Gasto Real Total],
        ALLSELECTED ( Gastos_2026[MES] ),
        Gastos_2026[MES] = MesActual - 1
    )

Variación Mes vs Mes Anterior % =
DIVIDE ( [Gasto Real Total] - [Gasto Mes Anterior], [Gasto Mes Anterior] )

Ranking Cuenta por Gasto =
RANKX ( ALLSELECTED ( Grupo_Cta_Ctble[NOM_CTBLE] ), [Gasto Real Total], , DESC )
```

**Uso:** el "Semáforo de Control" va perfecto en un visual **KPI** o como columna
condicional en las tablas; "Gasto Acumulado YTD" vs "Gasto Esperado a la Fecha"
es tu combo chart de control mientras no tengas presupuesto real.

---

## 2. Cuando tengas el presupuesto aprobado (recomendado)

Pide a Dirección un archivo con una fila por **Centro de Costo × Cuenta Contable × Mes**:

| GERENCIA | CTA_CTBLE | MES | Monto_Presupuestado |
|---|---|---|---|
| ... | ... | 1 | 50000 |

Impórtalo como tabla `Presupuesto`, relaciónala con `Grupo_Cta_Ctble` (por `CTA_CTBLE`)
y `Responsable_Cod_CC` (por el código de gerencia/CC), con cardinalidad muchos-a-uno,
igual que ya está relacionada `Gastos_2026`. Luego crea:

```dax
Ppto Total = SUM ( Presupuesto[Monto_Presupuestado] )

Variación vs Ppto $ = [Gasto Real Total] - [Ppto Total]

Variación vs Ppto % = DIVIDE ( [Variación vs Ppto $], [Ppto Total] )

% Ejecución Presupuestal = DIVIDE ( [Gasto Real Total], [Ppto Total] )

Ppto Disponible = [Ppto Total] - [Gasto Real Total]

Semáforo Ppto =
SWITCH (
    TRUE (),
    [% Ejecución Presupuestal] > 1.10, "🔴",
    [% Ejecución Presupuestal] > 0.95, "🟡",
    "🟢"
)
```

Estas reemplazan a las medidas "proxy" de la sección 1 una vez tengas datos reales.

---

## 3. Aplicar el tema visual (ya está en tu repo)

`tailwind_blue_theme.json` (rama `claude/power-bi-tailwind-theme-g53ms5`) ya trae
la paleta azul, bordes redondeados y sombra suave tipo tarjeta web.
**Vista → Temas → Examinar temas** → selecciona el archivo.

---

## 4. Rediseño página por página

### Resumen Ejecutivo
1. Agrupa los 6 slicers sueltos en un **panel lateral izquierdo** (ancho fijo ~200px,
   apilados verticalmente) en vez de dispersos — se ve más a "sidebar" de app web.
2. Convierte los slicers de `MES` y `GERENCIA` a estilo **"Tile" (botones/pills)**
   en la parte superior en vez de dropdown — Formato del slicer → Opciones → Estilo.
3. Fila de KPIs arriba: usa el visual **"Card (nuevo)"** (no el Card clásico) para
   `Gasto Real Total`, `Gasto Acumulado YTD`, `Proyección Anualizada de Gasto`,
   `% Avance del Año` y `Semáforo de Control` — permite ícono + variación + color
   condicional, se ve mucho más "web dashboard".
4. Cambia el `lineChart` de "Evolución Mensual del Gasto" por un **combo chart**:
   columnas = `Gasto Real Total`, línea = `Gasto Esperado a la Fecha` (línea de
   referencia/meta) — así se ve visualmente el control.
5. En el `barChart` "Gasto por Grupo de Cuenta": ordena descendente, activa
   etiquetas de datos, usa los colores del tema.

### Control por Cuenta
1. Igual que arriba: slicers a panel lateral.
2. En la tabla principal, activa **Data bars** (Formato → Elementos de celda →
   Barras de datos) sobre `Participación del Gasto %` y `Gasto Real Total` —
   lectura visual inmediata sin leer números.
3. Agrega **línea de referencia** (promedio) al `barChart` "Control por Cuenta"
   (Formato → Línea de referencia → Agregar).
4. Considera una visual de **Top N** (filtro Top 10 por `Gasto Real Total`) para
   que Dirección vea las cuentas críticas sin scroll.

### Responsables
1. Activa **Small multiples** en el `clusteredBarChart` "Gasto Real por Gerencia"
   partiendo por `GERENCIA` — genera automáticamente un panel comparativo tipo
   grid, muy usado en dashboards web modernos.
2. Agrega tabla/matriz `GERENCIA > COORDINACION > DESCRIPCION` con drill-down y
   Data bars.

### "Página 1" → renómbrala a "Detalle de Gastos"
1. Clic derecho en la pestaña → Cambiar nombre → "Detalle de Gastos".
2. Configura **drillthrough**: en esta página, agrega el campo `CTA_CTBLE` (o
   `GERENCIA`) al pozo "Drill through" (panel Visualizaciones) para poder hacer
   clic derecho sobre cualquier KPI/barra en las otras páginas → "Ver detalle".

---

## 5. Interactividad tipo app web (todas las páginas)

- **Barra de navegación superior**: crea 4 botones (uno por página) usando
  Insertar → Botones → En blanco, con acción "Ir a página". Desactiva la barra
  de pestañas nativa (Formato de página → Barra de navegación de página → Off)
  para que se vea como un app-shell, no como pestañas de Excel.
- **Botón "Restablecer filtros"**: crea un bookmark con el estado inicial de
  slicers y un botón con acción "Marcador".
- **Tooltips personalizados**: crea una página pequeña (150×80 px, marcada como
  "Tooltip page" en Formato de página) con una mini tendencia, y asígnala como
  tooltip de los visuales principales — efecto muy "web app" al pasar el mouse.
- Sincroniza los slicers de `MES` y `GERENCIA` entre páginas (pestaña Ver →
  Sincronizar Slicers) para que el filtro se mantenga al navegar.

---

## 6. Prioridad sugerida de implementación

1. Medidas DAX de la sección 1 (control con proxy).
2. Aplicar tema (`tailwind_blue_theme.json`).
3. Rediseño de "Resumen Ejecutivo" (la página que verá Dirección primero).
4. Navegación tipo web + drillthrough hacia "Detalle de Gastos".
5. Resto de páginas (Control por Cuenta, Responsables).
6. Cuando llegue el presupuesto real: medidas de la sección 2 y reemplazar los
   proxies.
