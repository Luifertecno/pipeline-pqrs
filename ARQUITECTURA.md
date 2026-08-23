# Arquitectura Técnica — Pipeline PQRS v1.0

---

## 1. Visión General

**Propósito:** Automatizar emparejamiento inteligente de órdenes médicas con referencias posteriores en sistemas de salud.

**Constraints:**
- Procesar 200K+ registros en <10 segundos
- Tolerar datos sucios (nulos, tipos mixtos, inconsistencias)
- Generar auditoría completa de excepciones
- Escalable a múltiples fuentes de datos

**Diseño:** Dual-phase ETL con profiling integrado

---

## 2. Fase 1: Consolidación

### 2.1 Entrada

- **Catálogo clasificatorio:** `N × servicios` (tabla de referencia)
- **Reportes mensuales:** `12+ × M registros/mes` (Excel/CSV)

### 2.2 Lógica

```
┌─────────────────────────────────────────────────────┐
│ FASE 1: CONSOLIDACIÓN                               │
├─────────────────────────────────────────────────────┤
│                                                     │
│ for archivo in reportes_mensuales:                 │
│   df_mes = read_excel(archivo)                     │
│   df_con_key = df_mes.merge(                       │
│       catálogo,                                    │
│       on='CODIGO_PROCEDIMIENTO',                   │
│       how='left'                                   │
│   )                                                │
│   resultados.append(df_con_key)                    │
│                                                     │
│ df_total = concat(resultados)                      │
│ df_total.to_csv('ordenes_consolidadas.csv')        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 2.3 Optimizaciones

| Técnica | Beneficio |
|---------|-----------|
| **LEFT JOIN** | Conserva órdenes sin clasificación para auditoría |
| **CSV output** | Supera límite de filas Excel (~1M) |
| **Iteración por mes** | Memoria: O(n/12) en lugar de O(n) |
| **Tracking origen** | Columna `_archivo_origen` para trazabilidad |

### 2.4 Salida

```
Archivo: ordenes_consolidadas.csv
├─ Columnas: DOCUMENTO_PACIENTE, FECHA_ORDEN, CODIGO_PROCEDIMIENTO, KEY, ...
├─ Filas: Todas las órdenes con KEY (clasificadas)
└─ Volumen: 100K-200K registros

Archivo: ordenes_sin_clasificacion.csv (excepciones)
├─ Órdenes donde CODIGO_PROCEDIMIENTO no existe en catálogo
└─ Requiere revisión manual del área de codificación
```

---

## 3. Fase 2: Emparejamiento + Métricas

### 3.1 Diseño de 7 Etapas

#### **Etapa 1: Carga de Órdenes**
- Lee CSV consolidado (potencialmente > 1 GB)
- Detecta automáticamente nombres de columna (maneja variaciones)
- Valida tipos: fecha → datetime, ID → string
- **Output:** DataFrame de órdenes normalizadas

#### **Etapa 2: Carga de Referencias**
- Lee archivo de referencias (Excel o CSV)
- Detección automática de columnas
- Normalización de tipos
- **Output:** DataFrame de referencias normalizadas

#### **Etapa 3: Preparación y Validación**
```
Para cada DataFrame:
  ├─ Detectar valores nulos por columna
  ├─ Convertir tipos (string, int, datetime)
  ├─ Validar fechas coherentes
  └─ Separar válidos de inválidos
  
Excepciones documentadas:
  ├─ DOCUMENTO_PACIENTE vacío
  ├─ CODIGO_PROCEDIMIENTO nulo
  ├─ FECHA_ORDEN inválida (no parseable)
  └─ Tipos incompatibles
```

**Estrategia:** No fallar todo por un registro malo. Capturar excepción + continuar.

#### **Etapa 4: Emparejamiento**

Vincula cada referencia con su orden médica de origen, validando integridad de datos y calculando indicadores de oportunidad.

**Output:** Pares orden-referencia validados con métricas de tiempo transcurrido.

**Tipos de excepción registrados:**

| Tipo | Descripción |
|------|-------------|
| `Referencia sin Orden` | No hay orden médica coincidente |
| `Orden sin Referencia` | Orden emitida sin referencia posterior |

**Características:**
- Optimizado para procesar 200K+ registros en <10 segundos
- Tolera datos fragmentados de múltiples fuentes
- Auditoría completa de no-matches
- Escalable a volúmenes altos

**Nota:** Lógica de negocio y detalles de implementación bajo NDA.

#### **Etapa 5: Cálculo de Métricas**

```
dias = df_pares['DIAS_TRANSCURRIDOS']

metricas = {
    'total_pares': len(df_pares),
    'promedio_dias': dias.mean(),
    'mediana_dias': dias.median(),        # Robusta ante outliers
    'p90_dias': dias.quantile(0.90),      # 90% de referencias ≤ X días
    'p95_dias': dias.quantile(0.95),      # 95% de referencias ≤ X días
}

indicadores = {
    'total_ordenes': len(df_ordenes_raw),
    'total_referencias': len(df_referencias_raw),
    'emparejamientos': len(df_pares),
    'cobertura_pct': len(df_pares) / len(ordenes_unicas) * 100,
}
```

**Interpretación:**
- `promedio_dias`: valor bruto (sensible a outliers)
- `mediana_dias`: valor central representativo
- `p90_dias`: 90% de casos ≤ X días → métrica operacional
- `cobertura_pct`: % de órdenes que generaron referencia → calidad de data

#### **Etapa 6: Validaciones Post-Emparejamiento**

Verifica integridad y coherencia de pares emparejados, detectando anomalías y datos inconsistentes.

**Output:** Dataset validado + reporte de anomalías.

**Nota:** Reglas de validación y criterios de aceptación bajo NDA.

#### **Etapa 7: Exportación**

```
Salida 1: resultados.csv
├─ Todas las columnas del par
├─ Delimitador: coma
├─ Encoding: UTF-8
└─ Volumen: 40K-50K pares típicamente

Salida 2: resultados.xlsx
├─ Hoja "Emparejadas"
├─ Columnas ordenadas para ejecutivos
├─ Formato: encabezados negros, datos blancos
└─ Volumen: 1-2 MB típicamente

Salida 3: excepciones.csv
├─ TODOS los registros que no emparejaron
├─ Columnas: [registro, tipo_excepcion, motivo]
├─ Volumen: 10-20% de referencias
└─ Crítico para auditoría

Salida 4: metricas_indicadores.xlsx
├─ Hoja "Metricas": promedio, mediana, p90, cobertura
├─ Hoja "Indicadores": totales y ratios
└─ Resumen ejecutivo (1 página)
```

---

## 4. Patrones y Decisiones Técnicas

### 4.1 Manejo de Errores

**Filosofía:** Fail-safe (no fallar por un registro, capturar excepción)

```
try:
    fecha_parsed = pd.to_datetime(fecha_raw)
except:
    excepciones.append({
        'REGISTRO_ID': record_id,
        'COLUMNA': 'FECHA_ORDEN',
        'VALOR': fecha_raw,
        'MOTIVO': 'Fecha no parseable'
    })
    continue  # Saltar este registro, continuar con el siguiente
```

### 4.2 Detección Automática de Columnas

**Problema:** Diferentes sistemas usan diferentes nombres
- Sistema A: `CEDULA_PACIENTE`
- Sistema B: `DOC_PAC`
- Sistema C: `DOCUMENTO`

**Solución:** Mapeos configurables

```python
COLUMNAS_ESPERADAS = {
    'DOCUMENTO_PACIENTE': ['CEDULA', 'DOC_PAC', 'DOCUMENTO', 'PATIENT_ID'],
    'FECHA_ORDEN': ['FECHA', 'FECHA_EMISION', 'ORDER_DATE'],
}

def detectar_columna(df, opciones):
    """Busca la columna de forma case-insensitive"""
    cols_lower = {c.lower(): c for c in df.columns}
    for opcion in opciones:
        if opcion.lower() in cols_lower:
            return cols_lower[opcion.lower()]
    raise ValueError(f"Columna no encontrada. Esperadas: {opciones}")
```

**Ventaja:** Adaptar a nuevas fuentes sin tocar código.

### 4.3 Profiling Integrado

Cada etapa mide:

```python
class ProfileManager:
    def iniciar_etapa(nombre):
        self.inicio = time.time()
        self.ram_antes = psutil.memory()
    
    def finalizar_etapa(nombre, registros):
        duracion = time.time() - self.inicio
        ram_despues = psutil.memory()
        velocidad = registros / duracion
        
        # Logs estructurados
        log(f"{nombre}: {duracion:.2f}s, {registros:,} reg/s")
```

**Salida:**

```
[14:23:45] ✓ Carga órdenes: 2.3 s, 43,478 reg/s
[14:23:47] ✓ Carga referencias: 1.1 s, 90,909 reg/s
[14:23:50] ✓ Emparejamiento: 2.1 s, 43,810 reg/s
[14:23:52] ✓ Validaciones: 0.8 s, 115,000 reg/s
[14:23:53] ✓ Exportación: 1.2 s
```

### 4.4 Escalabilidad: CSV Gigantes

**Problema:** CSV > 1 GB = no cabe en RAM

**Solución:** Chunking + reducción en memoria

```python
# Opción 1: Leer en chunks
for chunk in pd.read_csv(ruta, chunksize=10000):
    procesar(chunk)

# Opción 2: Usar Dask (escalable a Spark)
import dask.dataframe as dd
df = dd.read_csv(ruta)
```

**Actual:** Pandas + validación de memoria disponible antes de cargar.

### 4.5 Deduplicación Eficiente

Elimina duplicados por paciente + procedimiento, conservando la orden más relevante para el emparejamiento.

**Beneficio:** 30-40% reducción en comparaciones de emparejamiento.

**Nota:** Criterios y algoritmo de deduplicación bajo NDA.

---

## 5. Flujo de Datos

```
INPUT
  │
  ├─ ordenes_consolidadas.csv (200K+)
  └─ referencias_nurc.xlsx (200K+)
      │
      ├─ FASE 1: Consolidación
      │   └─ ordenes_consolidadas.csv
      │
      ├─ FASE 2: Emparejamiento
      │   ├─ Etapa 1-3: Carga + Prep
      │   ├─ Etapa 4: Match
      │   ├─ Etapa 5-7: Validación + Exportación
      │   │
      │   ├─ resultados.csv (pares emparejados)
      │   ├─ resultados.xlsx (ejecutivo)
      │   ├─ excepciones.csv (auditoría)
      │   └─ metricas_indicadores.xlsx (KPIs)
      │
      OUTPUT (calidad garantizada)
```

---

## 6. Casos de Uso

### 6.1 Sistema de Salud Ambulatorio
- Medición de oportunidad: ¿Cuánto tarda una cita desde la orden?
- KPI: p90 = 7 días (90% de citas dentro de 1 semana)

### 6.2 Sistema de Urgencias
- Priorización: órdenes de urgencia vs electivas
- Detección: referencias sin orden previa = órdenes perdidas

### 6.3 Auditoría de Calidad
- Excepciones = problemas en captura de datos
- Análisis: ¿Qué especialidades tienen peor cobertura?

### 6.4 Fintech/E-commerce (adaptado)
- Emparejamiento transacción → confirmación de pago
- Validación: qué pagos nunca se confirmaron

---

## 7. Limitaciones y Alternativas

### 7.1 Limitación: Emparejamiento 1-a-1

**Problema:** Una orden solo se puede usar una vez.  
**Razón:** Patrón de negocio (1 orden → 1 referencia)  
**Alternativa:** Permitir 1-a-N si el caso lo requiere

### 7.2 Limitación: Sin filtros de estado

**Problema:** Todas las referencias válidas entran en el emparejamiento.  
**Razón:** Archivo de referencias no incluye estado.  
**Alternativa:** Agregar filtro por estado si dato disponible

### 7.3 Limitación: RAM

**Problema:** CSV > 2 GB puede causar OOM.  
**Alternativa 1:** Usar Dask (escalable)  
**Alternativa 2:** SQL (PostgreSQL, DuckDB)  
**Alternativa 3:** Spark (para 100+ GB)

---

## 8. Versión

**Versión:** v1.0 · Estado: Producción

**Roadmap v1.1:**
- [ ] Soporte Dask para CSV > 2GB
- [ ] Exportación a Parquet
- [ ] Integración con PostgreSQL (incremental)

---

## 9. Contacto

**Preguntas sobre arquitectura, performance tuning, o adaptación a tu caso:**

Fernando Lizcano  
📧 ll.luifer@hotmail.com  
🔗 LinkedIn: [linkedin.com/in/fernandolizcano](https://www.linkedin.com/in/fernandolizcano)

*Disponible para consultoría de implementación.*
