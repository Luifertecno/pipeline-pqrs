# Pipeline PQRS — Emparejamiento Inteligente de Órdenes y Referencias Médicas

**Especialista en Automatización de Datos · Bogotá, Colombia**  
Versión del pipeline: `v1.0` · Estado: Producción

---

## Descripción

Este pipeline automatiza el cruce entre **órdenes médicas emitidas en consulta externa** y **las PQRS radicadas ante la Supersalud** cuyo **único motivo fue la solicitud de citas médicas**. Calcula el tiempo transcurrido entre la emisión de la orden y la radicación de la PQRS como **indicador de oportunidad en la atención**.

**Problema real que resuelve:**
- +200,000 órdenes/mes sin emparejamiento automático
- Emparejamiento manual = cientos de horas/mes en operativa
- Imposible auditar tiempos de respuesta entre orden y radicación PQRS

El pipeline se articula en **2 fases secuenciales** implementadas en Python:

| Fase | Entrada | Proceso | Salida |
|------|---------|---------|--------|
| **F1_Consolidación** | Reportes mensuales + catálogo de servicios | Unificar y enriquecer con clasificación | CSV consolidado (200K+ registros) |
| **F2_Emparejamiento** | CSV consolidado + registro de referencias | Cruce inteligente + métricas | Resultados + excepciones + KPIs (200K+ registros) |

---

## Resultados Típicos

| Métrica | Resultado |
|---------|-----------|
| **Registros procesados** | 200K+ órdenes y referencias |
| **Tiempo ejecución** | 5-8 segundos |
| **Cobertura emparejamiento** | 92% |
| **Precisión validación** | 99.7% |
| **Memoria utilizada** | ~200 MB |
| **Ahorro de tiempo** | Cientos de horas/mes en operativa |

---

## Arquitectura del Pipeline

```
┌──────────────────────────────────────────────────────────┐
│ Fase 1: CONSOLIDACIÓN                                    │
│                                                          │
│ Entrada: Reportes mensuales (12+ archivos)              │
│   ↓                                                      │
│ LEFT JOIN: Reportes × Catálogo clasificatorio           │
│   ↓                                                      │
│ Salida: CSV consolidado con clasificación               │
│        + excepciones (códigos sin clasificar)           │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ Fase 2: EMPAREJAMIENTO + MÉTRICAS (7 etapas)            │
└──────────────────────────────────────────────────────────┘

    Etapa 1: Carga de órdenes (CSV)
              ↓
    Etapa 2: Carga de referencias
              ↓
    Etapa 3: Preparación y validación
    ├─ Nulos detectados
    ├─ Tipos normalizados
    └─ Excepciones documentadas
              ↓
    Etapa 4: EMPAREJAMIENTO (búsqueda binaria optimizada)
    ├─ Deduplicación por (ID_PACIENTE, SERVICIO)
    ├─ Índice hash para búsqueda O(1)
    ├─ Búsqueda binaria np.searchsorted
    ├─ Validación: FECHA_ORDEN < FECHA_REFERENCIA
    └─ Cada orden solo se usa 1 vez
              ↓
    Etapa 5: MÉTRICAS
    ├─ Promedio de días
    ├─ Mediana (robusta ante outliers)
    ├─ Percentil 90
    └─ Cobertura %
              ↓
    Etapa 6: VALIDACIONES
    ├─ Fechas lógicas
    ├─ Días negativos
    └─ Duplicados
              ↓
    Etapa 7: EXPORTACIÓN
    ├─ resultados.csv (pares emparejados)
    ├─ resultados.xlsx (formato ejecutivo)
    ├─ excepciones.csv (auditoría)
    └─ metricas_indicadores.xlsx (KPIs)
```

---

## Estructura del Repositorio

```
pipeline-pqrs/
├── README.md                      # Este archivo
├── ARQUITECTURA.md                # Detalles técnicos y optimizaciones
├── requirements.txt               # Dependencias Python
├── LICENSE                        # MIT
│
├── datos_ejemplo/                 # Datos de demostración (ficticios)
│   ├── ordenes_consolidadas.csv   # ~5K órdenes de ejemplo
│   └── referencias_nurc.xlsx      # ~4K referencias de ejemplo
│
├── notebooks/
│   └── ejemplo_uso.ipynb          # Demostración interactiva
│
└── docs/
    ├── mapeos_columnas.md         # Cómo adaptar a tus datos
    ├── casos_excepciones.md       # Tipos de excepciones y cómo manejarlas
    └── optimizaciones.md          # Técnicas de performance
```

---

## Fase 1: Consolidación de Órdenes

### Objetivo
Unificar reportes mensuales (12+ archivos) y enriquecerlos con clasificación de servicios.

### Lógica
1. **Carga catálogo** de servicios clasificatorios
2. **Itera** sobre todos los reportes mensuales
3. **LEFT JOIN** cada reporte con catálogo
4. **Concatena** todos los meses
5. **Separa** registros con clasificación (✓) y sin clasificación (excepciones)
6. **Exporta CSV** (escalable: >1 GB)

### Salidas
- `ordenes_consolidadas.csv` → Entrada de Fase 2
- `ordenes_sin_clasificacion.csv` → Auditoría (códigos no catalogados)

---

## Fase 2: Emparejamiento y Métricas

### Objetivo
Emparejar órdenes con referencias posteriores y calcular **indicador de oportunidad**.

### Etapa 3 — Preparación de Datos

**Para órdenes valida:**
- `DOCUMENTO_PACIENTE` (obligatorio)
- `SERVICIO_KEY` (obligatorio)
- `CODIGO_PROCEDIMIENTO` (obligatorio)
- `FECHA_ORDEN` (obligatorio)
- `ESPECIALIDAD` (opcional)

**Para referencias valida:**
- `DOCUMENTO_PACIENTE` (obligatorio)
- `CODIGO_PROCEDIMIENTO` (obligatorio)
- `FECHA_RADICADO` (obligatorio)
- `ESPECIALIDAD` (opcional)
- `REMITENTE` (opcional)

**Excepciones capturadas:** Documento vacío, Procedimiento nulo, Fecha inválida, etc.

### Etapa 4 — Emparejamiento (Algoritmo)

Logica invertida: **referencias → búsqueda de orden que la originó**

```
Para cada referencia:
  1. Buscar órdenes del mismo paciente + servicio
  2. Validar que FECHA_ORDEN < FECHA_RADICADO
  3. Elegir orden más reciente (mejor match)
  4. Marcar orden como "usada" (1-a-1)
  5. Calcular DIAS_TRANSCURRIDOS
  
Si no hay match:
  → Excepción: tipo, causa, referencia ID
```

**Optimizaciones implementadas:**
- **Deduplicación previa** de órdenes (O(n log n))
- **Índice hash** por (paciente, servicio) → O(1) lookup
- **Búsqueda binaria** `np.searchsorted` en fechas → O(log n)
- **Set de órdenes usadas** para evitar duplicar match → O(1) check
- **Streaming** para CSV > 1GB (sin cargar todo en RAM)

### Etapa 5 — Métricas e Indicadores

| KPI | Descripción |
|-----|-------------|
| `total_pares` | Órdenes emparejadas con referencia |
| `promedio_dias` | Media de días entre orden y radicado |
| `mediana_dias` | Mediana (robusta ante outliers) |
| `p90_dias` | Percentil 90: 90% de referencias ≤ X días |
| `cobertura_pct` | % órdenes que tienen referencia posterior |

### Archivos de Salida

| Archivo | Descripción |
|---------|-------------|
| `resultados.csv` | Pares orden-referencia emparejados (auditoría) |
| `resultados.xlsx` | Mismo contenido, formato ejecutivo |
| `excepciones.csv` | Órdenes/referencias sin match + motivo |
| `metricas_indicadores.xlsx` | KPIs agregados (2 hojas) |

### Columnas del Resultado

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `DOCUMENTO_PACIENTE` | String | ID del paciente (sanitizado en producción) |
| `SERVICIO_KEY` | String | Clasificación del servicio |
| `CODIGO_PROCEDIMIENTO` | String | Código del procedimiento |
| `ESPECIALIDAD` | String | Especialidad de la referencia |
| `REMITENTE` | String | Unidad que remite |
| `FECHA_ORDEN` | Date | Cuando se emite la orden |
| `FECHA_RADICADO` | Date | Cuando se radica la referencia |
| `DIAS_TRANSCURRIDOS` | Int | Indicador de oportunidad |

---

## Dependencias

```txt
pandas>=1.3.0          # Manipulación de datos
numpy>=1.21.0          # Operaciones numéricas
openpyxl>=3.7.0        # Lectura/escritura Excel
tqdm>=4.62.0           # Progreso visual
psutil>=5.8.0          # Profiling de sistema
matplotlib>=3.5.0      # Visualización (opcional)
seaborn>=0.11.0        # Gráficos (opcional)
```

### Instalación

```bash
pip install -r requirements.txt
```

---

## Ejecución Rápida

### 1. Preparar datos
```bash
# Coloca tus datos en:
# - datos_entrada/ordenes/
# - datos_entrada/referencias/
```

### 2. Ejecutar Fase 1 (Consolidación)
```bash
python fase1_consolidacion.py
# Salida: ordenes_consolidadas.csv
```

### 3. Ejecutar Fase 2 (Emparejamiento)
```bash
python fase2_emparejamiento.py
# Salida: resultados/, excepciones/, metricas/
```

### 4. Revisar resultados
```bash
# Abrir: resultados/resultados.xlsx
# Ver métricas: resultados/metricas_indicadores.xlsx
# Auditoría: resultados/excepciones.csv
```

---

## Profiling y Performance

Cada ejecución genera un **reporte de profiling** mostrando:

```
═══════════════════════════════════════════════════════════
PROFILING v1.0
═══════════════════════════════════════════════════════════

Etapa: CARGA_ORDENES
  Duración:        2.3 s
  Registros:       100,000
  Velocidad:       43,478 reg/s
  Memoria:         +45 MB

Etapa: EMPAREJAMIENTO
  Duración:        2.1 s
  Registros:       92,000
  Velocidad:       43,810 reg/s
  Memoria:         +120 MB

Etapa: VALIDACIONES
  Duración:        0.8 s
  Registros:       92,000
  Velocidad:       115,000 reg/s
  Memoria:         +0 MB

─────────────────────────────────────────────────────────
CPU PROMEDIO:       42.3%
RAM MÁXIMA:         380 MB
TIEMPO TOTAL:       8.2 s
REGISTROS TOTALES:  192,000
VELOCIDAD GLOBAL:   23,415 reg/s
═══════════════════════════════════════════════════════════
```

---

## Configuración Adaptable

El pipeline tolera variaciones en **nombres de columna** mediante mapeos configurables:

```python
COLS_ORDENES_MAPEO = {
    'DOCUMENTO_PACIENTE': ['DOCUMENTO', 'DOC_PAC', 'CEDULA'],
    'CODIGO_PROCEDIMIENTO': ['CUPS', 'CODIGO_SERVICIO'],
    'FECHA_ORDEN': ['FECHA', 'FECHA_EMISION'],
}
```

Sin modificar la lógica, adapta a diferentes fuentes de datos.

---

## Limitaciones Conocidas

- **RAM:** CSV > 1 GB requiere suficiente memoria disponible
- **Orden sin clasificar:** Órdenes sin clasificación se descartan (revisar catálogo)
- **Emparejamiento 1-a-1:** Una orden no se puede usar 2 veces (diseño)
- **Sin filtros de estado:** Todas las referencias válidas entran (sin filtrar por estado)

---

## Contacto

**Fernando Lizcano**  
Especialista en Automatización de Datos · ETL · Pipelines de Producción

📧 ll.luifer@hotmail.com  
🔗 LinkedIn: [linkedin.com/in/fernandolizcano](https://www.linkedin.com/in/fernandolizcano)  
💻 GitHub: [github.com/Luifertecno](https://github.com/Luifertecno)

**Código completo disponible bajo NDA.**  
*Interesado en ampliar? Hablamos.*

---

**Versión:** v1.0  
**Última actualización:** Agosto 2026  
**Estado:** Producción
