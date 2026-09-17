# Observatorio Nacional del Mercado Eléctrico Colombiano

**Introducción a la Analítica de Negocios** · Línea de énfasis en Analítica
**Departamento de Ingeniería Industrial** · Universidad de Antioquia
**Proyecto integrador — Entrega 2**

**Integrantes:** Juan Esteban Diosa Sanmartín · Juliana Hurtado Mosquera · Daniel Alejandro Osorio

---

## Problemática, preguntas analíticas y periodicidad

El observatorio integra información del operador del mercado eléctrico (XM), de la tasa de cambio oficial (Superintendencia Financiera / Banco de la República) y del índice de precios al consumidor (DANE) para entender, a nivel nacional y diario, cómo se relacionan la demanda, el precio de bolsa y la disponibilidad hídrica de energía en Colombia, y cómo el calendario y el contexto macroeconómico se relacionan con esas variaciones.

Preguntas analíticas:

- ¿Cómo se relaciona el nivel de embalses (disponibilidad hídrica) con el precio de bolsa nacional de energía?
- ¿Cómo varía la demanda nacional de energía entre días hábiles, fines de semana y festivos?
- ¿Cómo se relaciona el precio de bolsa con la tasa de cambio (TRM) y con el índice de precios al consumidor (IPC) de energía?

**Periodicidad recomendada:** las variables de demanda, precio de bolsa, volumen útil de embalses y TRM tienen granularidad diaria, por lo que se recomienda una **extracción diaria (proceso batch automatizado)** para mantener un tablero actualizado en tiempo real. El IPC de energía se publica mensualmente, por lo que su actualización se programa una vez al mes, propagando el valor vigente a cada día correspondiente. El calendario de festivos se regenera una vez al año.

## Proceso de extracción automatizada

El proceso se implementó como un pipeline en Python (notebook incluido en este repositorio), organizado en módulos independientes con manejo de excepciones y reintentos automáticos, de forma que el fallo de una fuente no detiene el proceso completo. Si esto ocurre, basta con ajustar el rango de fechas en la celda de configuración del notebook y volver a ejecutar. Todo el flujo es reproducible sin intervención manual:

- **XM / SIMEM:** se usa la librería oficial `pydataxm` (clase `ReadDB`), que expone la API pública de XM sin usuario ni llave. El catálogo de métricas se descarga una sola vez por ejecución y se reutiliza; los identificadores de métrica se localizan por búsqueda de texto sobre ese catálogo (probando varias frases candidatas en cascada), ya que su nomenclatura puede cambiar entre actualizaciones. Todas las consultas se hacen a nivel de Sistema (nacional), sin desagregar por planta ni región.
- **TRM:** se consulta el dataset oficial de la Tasa Representativa del Mercado en el portal de datos abiertos del Gobierno (`datos.gov.co`, dataset `mcec-87by`); las columnas de fecha y valor se descubren automáticamente antes de filtrar por rango de fechas.
- **IPC de energía:** se descarga directamente el archivo mensual de anexos del DANE y se ubica la fila de la división "Alojamiento, Agua, Electricidad, Gas y Otros Combustibles". El DANE cambió la estructura de carpetas de estos archivos durante 2024, por lo que se prueban dos patrones de ruta en cascada; si el archivo de un mes puntual no existe, ese mes se omite y el proceso continúa con los demás.
- **Calendario de festivos:** se genera con la librería estándar de festivos de Colombia (variable auxiliar, no es una fuente de datos externa).

## Diseño y estructura de la base maestra

La base maestra se construye a nivel nacional-diario, usando como llave única de integración la **fecha**. Matriz de homologación de variables:

| Variable original | Fuente | Nombre homologado | Unidad | Granularidad |
|---|---|---|---|---|
| Demanda comercial (kWh) | XM / SIMEM | `demanda_gwh` | GWh | Diaria |
| Precio de bolsa nacional | XM / SIMEM | `precio_bolsa_cop_kwh` | COP/kWh | Diaria |
| Volumen útil de embalses | XM / SIMEM | `volumen_util_embalses_kwh` | kWh | Diaria |
| Tasa de cambio | Superfinanciera / Banrep | `trm` | COP/USD | Diaria |
| Variación anual IPC división vivienda-energía | DANE | `ipc_var_anual_energia` | % | Mensual |
| Tipo de fecha | Librería `holidays` (CO) | `tipo_dia` | Categórica | Diaria |

Demanda, precio de bolsa, volumen de embalses, TRM y el calendario de festivos se unen de forma directa por la fecha exacta. El IPC de energía, al ser mensual, se integra por el periodo año-mes, propagando su valor a cada día del mes correspondiente. El resultado se exporta en formato `.parquet` (tipado y comprimido) y en `.csv` (para inspección rápida).

## Cómo ejecutar

1. Abrir el notebook en Jupyter o Google Colab.
2. Ajustar `FECHA_INICIO` y `FECHA_FIN` en la celda de configuración si se desea otro rango.
3. Ejecutar todas las celdas en orden. Al final se imprime un resumen con el estado de cada fuente (`OK` / `FALLÓ`) y el tiempo total de ejecución.
4. La base maestra queda guardada en `datos_procesados/`.

## Limitaciones

- La disponibilidad hídrica se aproxima con el volumen útil de embalses (agregado nacional); no se clasifica la generación planta por planta.
- El IPC de energía usa la división "Alojamiento, Agua, Electricidad, Gas y Otros Combustibles" del DANE (categoría más cercana disponible, no exclusiva de electricidad).
- La TRM no se publica de forma idéntica todos los días del calendario, por lo que puede haber valores nulos puntuales.
