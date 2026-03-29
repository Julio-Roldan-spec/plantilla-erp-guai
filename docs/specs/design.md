# Modulo Plantilla — Gestion de Operarios en Plantilla + Orquestador de Asignacion Hibrida

**ERP guAI — Ampliacion para Grupo CuidaCasa**
**Fecha:** 2026-03-29
**Estado:** Aprobado por usuario

---

## 1. Vision y Proposito

El modulo Plantilla amplia el ERP guAI para gestionar operarios contratados en plantilla (empleados) junto al modelo actual de operarios autonomos, con un Orquestador Inteligente que decide automaticamente cuando asignar plantilla vs autonomo optimizando coste, disponibilidad y zona.

**Problema que resuelve:**
- No existe capacidad de gestionar operarios contratados (solo autonomos)
- No hay forma de comparar coste real plantilla vs autonomo por expediente
- No hay datos para decidir donde es rentable contratar empleados propios
- La asignacion no optimiza la mezcla de recursos disponibles

**Enfoque arquitectonico:** Capa Unificada con Orquestador Inteligente. El Orquestador es el cerebro central que recibe todo servicio nuevo y decide la estrategia optima (plantilla, autonomo o mixta) basandose en reglas configurables y analisis de coste.

**Cobertura inicial:** Delegaciones propias (Alicante, Valencia, Castellon). Expansion configurable por zona.

**Facturacion a compania aseguradora:** Mismo baremo que autonomo. El margen = baremo cobrado - coste interno del empleado.

**Relacion con Red guAI:** Motores separados. Red guAI sigue gestionando autonomos de forma independiente. El Orquestador decide cuando derivar a Red guAI.

---

## 2. Estructura del Modulo

### 2.1 Nuevo modulo en el sidebar

```
Plantilla
  ├── Empleados                    (ficha laboral del empleado)
  ├── Partes de Trabajo            (registro horas + materiales por expediente)
  ├── Jornada y Fichajes           (entrada/salida, cumplimiento legal RD 8/2019)
  ├── Turnos y Cuadrantes          (planificacion semanal/mensual)
  ├── Equipamiento Asignado        (herramientas, EPI, vehiculo empresa)
  └── Dashboard de Rentabilidad    (plantilla vs autonomo, ocupacion, simulador)
```

### 2.2 Cambios transversales en modulos existentes

- **Asistencia > Expedientes**: Al asignar servicio, el Orquestador decide si va a Plantilla o a Red guAI
- **Facturacion**: Nuevo concepto "Coste interno" para servicios de plantilla
- **Configuracion**: Nuevo sub-apartado "Reglas de Asignacion Hibrida"
- **Planificacion**: Vista unificada con empleados y autonomos
- **Informes**: 5 nuevos reportes de rentabilidad comparada

### 2.3 Modelos laborales soportados (configurables por delegacion)

1. **Multidisciplinar**: Un empleado cubre varios gremios (perfil multioficio)
2. **Especializado**: Un empleado tiene 1-2 gremios fijos
3. **Equipos/Cuadrillas**: Jefe de equipo + 1-2 operarios con especialidades complementarias

---

## 3. Ontologia del Dominio

### 3.1 Entidades nuevas (10)

| Entidad | Descripcion |
|---------|-------------|
| **Empleado** | Operario contratado en plantilla (datos laborales, no mercantiles) |
| **ContratoLaboral** | Tipo contrato, jornada, fecha inicio/fin, categoria profesional |
| **ParteTrabajo** | Registro de horas + materiales por expediente/servicio |
| **Fichaje** | Entrada/salida diaria (cumplimiento RD 8/2019) |
| **Turno** | Definicion de turno (manana/tarde/noche/guardia/partido) |
| **Cuadrante** | Planificacion semanal/mensual de turnos por empleado |
| **Cuadrilla** | Grupo de empleados con jefe de equipo (opcional, para modelo equipos) |
| **EquipamientoAsignado** | Herramienta, EPI o vehiculo asignado a un empleado |
| **CosteInterno** | Coste/hora calculado del empleado (salario + SS + equipamiento + vehiculo) |
| **DecisionOrquestador** | Registro trazable de cada decision plantilla vs autonomo |

### 3.2 Entidades existentes ampliadas

| Entidad ERP | Cambio |
|-------------|--------|
| **Expediente** | Nuevo campo `tipo_recurso_asignado` (PLANTILLA / AUTONOMO / MIXTO) |
| **Servicio** (dentro de expediente) | Nuevo campo `empleado_id` (alternativo a `operario_id`) |
| **Factura a compania** | Campo `coste_interno` para calcular margen real |
| **Baremo de operario** | Nuevo baremo de tipo "plantilla" = coste/hora interno |

### 3.3 Relaciones clave

```
Empleado 1──N ParteTrabajo N──1 Expediente/Servicio
Empleado 1──N Fichaje
Empleado 1──1 ContratoLaboral (activo)
Empleado N──M Especialidad (gremios que domina)
Empleado N──M ZonaCobertura (zonas donde opera)
Empleado N──1 Cuadrilla (opcional)
Cuadrilla 1──1 Empleado (jefe de equipo)
Empleado 1──N EquipamientoAsignado
Empleado 1──1 CosteInterno (calculado mensual)
DecisionOrquestador N──1 Expediente
```

### 3.4 Reglas ontologicas inmutables

| Regla | Descripcion |
|-------|-------------|
| R-PLA-01 | No asignar empleado fuera de su jornada laboral sin registro de horas extra |
| R-PLA-02 | No asignar empleado que supere 10h diarias (limite legal) |
| R-PLA-03 | No asignar empleado en periodo de vacaciones, baja o ausencia |
| R-PLA-04 | No asignar empleado a gremio que no tiene en su ficha de especialidades |
| R-PLA-05 | No asignar empleado fuera de sus zonas de cobertura asignadas |
| R-PLA-06 | Todo servicio de plantilla genera ParteTrabajo obligatorio |
| R-PLA-07 | Toda decision del Orquestador genera DecisionOrquestador con trazabilidad |
| R-PLA-08 | El fichaje diario es obligatorio — empleado sin fichar no puede recibir asignaciones |
| R-PLA-09 | Coste interno se recalcula mensualmente (salario + SS + prorratas) |
| R-PLA-10 | Si cuadrilla, la asignacion es al equipo completo, no a miembros individuales |
| R-PLA-11 | Descanso minimo 12h entre fin de jornada y siguiente asignacion |
| R-PLA-12 | Empleado con contrato temporal: alerta 30 dias antes de vencimiento |

---

## 4. Ficha del Empleado

### 4.1 Datos generales (cabecera)

| Campo | Tipo | Notas |
|-------|------|-------|
| Nombre completo | string | Obligatorio |
| DNI/NIE | string | Obligatorio, cifrado en BD |
| N. Seguridad Social | string | Obligatorio, cifrado en BD |
| Categoria profesional | enum | Oficial 1a, Oficial 2a, Ayudante, Jefe equipo |
| Modelo laboral | enum | Multidisciplinar / Especializado / Equipo |
| Delegacion | enum | ALC / VLC / CAS (ampliable) |
| Telefono 1, 2 | string | |
| Email | string | |
| Direccion | string | Para calculo de desplazamiento |
| Codigo postal | string | |
| Fecha alta empresa | date | |
| Activo | boolean | |
| Foto | imagen | Opcional |

### 4.2 Pestanas

| Pestana | Contenido |
|---------|-----------|
| **CONTRATO** | Tipo (indefinido/temporal/obra), jornada (completa/parcial), horas semanales, fecha inicio/fin, convenio, salario bruto anual |
| **ESPECIALIDADES** | Gremios que domina (misma tabla que operarios autonomos). Nivel: Principal / Secundario |
| **ZONAS** | Zonas de cobertura asignadas (misma logica que operarios) |
| **JORNADA** | Vista de fichajes del mes, horas trabajadas vs contratadas, horas extra acumuladas |
| **PARTES DE TRABAJO** | Lista de partes por expediente: fecha, expediente, servicio, horas, materiales, coste |
| **TURNOS** | Cuadrante mensual del empleado, turno asignado por dia |
| **EQUIPAMIENTO** | Herramientas, EPIs, vehiculo empresa asignado. Con fecha entrega y estado |
| **CUADRILLA** | Si modelo equipo: miembros del equipo, jefe, especialidades cubiertas |
| **COSTE** | Desglose coste/hora: salario + SS (33%) + prorrata equipamiento + prorrata vehiculo + EPI. Margen medio por expediente |
| **CALENDARIO** | Vista anual: vacaciones, bajas, festivos, formacion |

### 4.3 Diferencias clave vs ficha de operario autonomo

| Concepto | Operario autonomo | Empleado plantilla |
|----------|-------------------|--------------------|
| Identificacion fiscal | CIF / Razon Social | DNI + N.SS |
| Facturacion | Presenta factura propia | No factura, genera parte de trabajo |
| Disponibilidad | Libre (el autonomo gestiona) | Vinculada a jornada/turno/calendario |
| Coste | Precio baremo operario | Coste/hora calculado (salario+SS+...) |
| Datos bancarios | Para pagarle facturas | No necesario (nomina por asesoria) |
| Sub-operarios | Puede tener | No aplica (la cuadrilla es otra cosa) |
| Conceptos facturables | Si (desplazamiento, urgencia...) | No (coste fijo interno) |

---

## 5. Orquestador de Asignacion Hibrida

### 5.1 Flujo de decision

```
Nuevo servicio en expediente (gremio + zona + urgencia)
    |
    v
FASE 1: Filtrado de Plantilla
    |
    +-- HAY candidatos --> FASE 2: Evaluar coste vs autonomo
    |                          |
    |                          v
    |                      FASE 3: Decision
    |                          |
    |                     +----+----+
    |                     |         |
    |                     v         v
    |                  Plantilla  Red guAI
    |
    +-- NO hay candidatos --> Derivar a Red guAI (autonomos)
    |
    v
FASE 4: Registro de decision (DecisionOrquestador)
```

### 5.2 FASE 1 — Filtrado de candidatos plantilla

Aplica reglas R-PLA-01 a R-PLA-08 como filtros eliminatorios:

| Filtro | Criterio |
|--------|----------|
| Especialidad | Empleado tiene el gremio requerido (principal o secundario) |
| Zona | Empleado cubre la zona del siniestro |
| Jornada | Dentro de horario laboral o disponible para horas extra |
| Limite legal | No supera 10h diarias ni viola descanso 12h |
| Ausencias | No esta de vacaciones, baja o ausencia |
| Fichaje | Ha fichado hoy (R-PLA-08) |
| Carga | No supera umbral maximo de expedientes activos |

### 5.3 FASE 2 — Calculo de coste comparado

```
Coste_plantilla = horas_estimadas x coste_hora_empleado + materiales_estimados + desplazamiento_km
Coste_autonomo  = baremo_compania x unidades_estimadas (lo que pagariamos al autonomo)
Ingreso_cia     = baremo_compania x unidades_estimadas (lo que cobra la aseguradora)

Margen_plantilla = Ingreso_cia - Coste_plantilla
Margen_autonomo  = Ingreso_cia - Coste_autonomo
Delta_margen     = Margen_plantilla - Margen_autonomo
```

### 5.4 FASE 3 — Reglas de decision (configurables por delegacion)

| Regla | Condicion | Accion |
|-------|-----------|--------|
| RO-01 | `Delta_margen > 0` Y empleado disponible | **Plantilla** (mas rentable) |
| RO-02 | `Delta_margen < 0` pero `< umbral_negativo` | **Plantilla** si ocupacion < 70% (aprovechar capacidad ociosa) |
| RO-03 | `Delta_margen < umbral_negativo` | **Red guAI** (autonomo mas barato) |
| RO-04 | Urgencia fuera de horario laboral | **Red guAI** (autonomo con guardia) |
| RO-05 | Empleado disponible pero a >45 min del siniestro | Evaluar: si autonomo a <20 min -> **Red guAI** |
| RO-06 | Servicio requiere cuadrilla pero solo hay 1 empleado | **Red guAI** o **Mixto** |
| RO-07 | Ocupacion plantilla zona > 90% | **Red guAI** + alerta de saturacion |
| RO-08 | Ocupacion plantilla zona < 40% | Forzar **Plantilla** (evitar infrautilizacion) |

### 5.5 FASE 4 — Registro de decision (trazabilidad completa)

Cada decision se guarda en `DecisionOrquestador`:
```json
{
  "expediente_id": "...",
  "servicio_id": "...",
  "gremio": "fontaneria",
  "zona": "Valencia centro",
  "fecha_hora": "2026-03-29T10:30:00",
  "candidatos_plantilla": ["..."],
  "candidato_seleccionado": "...",
  "tipo_decision": "PLANTILLA",
  "regla_aplicada": "RO-01",
  "razon": "Delta margen +47 EUR, empleado a 12 min",
  "coste_plantilla_estimado": 185.00,
  "coste_autonomo_estimado": 232.00,
  "delta_margen": 47.00,
  "ocupacion_plantilla_zona": 68
}
```

**Override manual:** El tramitador puede forzar la decision del Orquestador con motivo obligatorio.

---

## 6. Dashboard de Rentabilidad y Simulador

### 6.1 KPIs principales (cabecera)

| KPI | Calculo | Objetivo |
|-----|---------|----------|
| **Ahorro mensual plantilla** | Suma de `delta_margen` positivo del mes | Ver cuanto se ahorra vs todo-autonomos |
| **Ocupacion plantilla** | Horas productivas / Horas contratadas x 100 | Objetivo: 70-85% |
| **Coste medio/servicio plantilla** | Total coste interno / Servicios completados | Comparar con coste medio autonomo |
| **% servicios cubiertos por plantilla** | Servicios plantilla / Total servicios x 100 | Medir penetracion |

### 6.2 Graficos interactivos

- **Margen comparado mensual**: Barras apiladas — margen real plantilla vs margen que habria sido con autonomo. Por delegacion.
- **Ocupacion por empleado**: Heatmap semanal — cada fila un empleado, cada celda un dia, color por % ocupacion
- **Distribucion por gremio**: Donut chart — que % de cada gremio cubre plantilla vs autonomo
- **Evolucion temporal**: Linea de tendencia — ahorro acumulado mes a mes

### 6.3 Simulador de contratacion

**Inputs:**

| Parametro | Tipo | Ejemplo |
|-----------|------|---------|
| Zona/delegacion | selector | Valencia |
| Gremio | selector | Fontaneria |
| N. empleados nuevos | numero | 2 |
| Jornada | selector | Completa / Parcial |
| Salario bruto anual estimado | numero | 24.000 EUR |

**Outputs (calculados con datos historicos 6 meses):**

- Expedientes/mes que pasarian de autonomo a plantilla
- Ahorro mensual estimado
- Coste mensual nuevos empleados (salario+SS+equip)
- Punto de equilibrio (ocupacion minima)
- Ocupacion estimada (basada en volumen historico)
- Resultado neto mensual estimado
- ROI anualizado
- Recomendacion: RENTABLE / NO RENTABLE / MARGINAL

### 6.4 Alertas proactivas

| Alerta | Trigger | Destinatario |
|--------|---------|-------------|
| **Saturacion zona** | Ocupacion > 90% durante 5 dias consecutivos | Responsable delegacion |
| **Infrautilizacion** | Ocupacion < 40% durante 10 dias | Direccion operaciones |
| **Oportunidad contratacion** | >20 servicios/mes en zona sin plantilla donde margen seria positivo | Direccion |
| **Vencimiento contrato** | Empleado temporal a 30 dias de fin | RRHH + Responsable |
| **Desequilibrio gremio** | Un gremio tiene >80% autonomo en zona con plantilla disponible | Responsable delegacion |
| **Horas extra excesivas** | Empleado acumula >20h extra/mes durante 2 meses | RRHH |

---

## 7. Partes de Trabajo y Control de Jornada

### 7.1 Parte de Trabajo

Cada vez que un empleado realiza un servicio en un expediente:

| Campo | Tipo | Notas |
|-------|------|-------|
| Expediente | ref | Vinculado al expediente del ERP |
| Servicio | ref | Servicio especifico (fontaneria, electricidad...) |
| Empleado | ref | Quien realizo el trabajo |
| Fecha | date | |
| Hora inicio | time | Llegada al domicilio |
| Hora fin | time | Salida del domicilio |
| Horas efectivas | decimal | Calculado automaticamente |
| Desplazamiento km | decimal | Desde ubicacion anterior o delegacion |
| Estado trabajo | enum | Completado / Parcial / Requiere revisita / Materiales pendientes |
| Materiales utilizados | lista | Nombre + cantidad + coste unitario |
| Coste materiales | decimal | Calculado |
| Coste total servicio | decimal | (Horas x coste/hora) + materiales + desplazamiento |
| Observaciones | texto | |
| Fotos | adjuntos | Antes/despues |
| Firma asegurado | imagen | Conformidad del cliente |

**Flujo:**
```
Empleado llega al domicilio
  -> Registra hora inicio
  -> Realiza el trabajo
  -> Registra materiales usados
  -> Toma fotos del resultado
  -> Registra hora fin
  -> Asegurado firma conformidad
  -> Parte se vincula al expediente
  -> Coste interno se calcula y alimenta al Dashboard
```

### 7.2 Control de Jornada (fichajes)

Cumplimiento del RD 8/2019:

| Campo | Tipo |
|-------|------|
| Empleado | ref |
| Fecha | date |
| Hora entrada | time |
| Hora salida | time |
| Horas totales | decimal (calculado) |
| Tipo jornada | enum: Normal / Horas extra / Guardia / Festivo |
| Metodo fichaje | enum: App movil / ERP web / Tablet delegacion |
| Ubicacion GPS | coordenadas (opcional) |

**Vista mensual de jornada** (pestana JORNADA de la ficha empleado):
- Calendario con horas diarias
- Resumen: horas contratadas vs trabajadas vs extra
- Horas productivas (en expedientes) vs improductivas (desplazamiento, espera)
- Porcentaje de ocupacion

### 7.3 Turnos y Cuadrantes

| Turno | Horario | Uso tipico |
|-------|---------|-----------|
| Manana | 07:00-15:00 | Estandar para mayoria de siniestros |
| Tarde | 14:00-22:00 | Citas de tarde y urgencias vespertinas |
| Partido | 08:00-13:00 + 16:00-19:00 | Adaptado a disponibilidad de asegurados |
| Guardia | 22:00-07:00 | Solo si se cubre urgencias nocturnas con plantilla |

**Cuadrante mensual**: Vista de planificacion por empleado, con turno asignado por dia. El Orquestador consulta este cuadrante en tiempo real.

---

## 8. Integracion con Modulos Existentes

### 8.1 Expedientes — Asignacion de servicios

Flujo actual: crear servicio -> seleccionar operario autonomo.
Flujo nuevo: crear servicio -> Orquestador decide automaticamente -> asigna empleado O deriva a Red guAI.

En la vista del expediente:
- Badge visual: "Plantilla" o "Autonomo" por servicio
- Si plantilla: nombre empleado + parte de trabajo vinculado
- Si autonomo: igual que ahora
- Override manual del tramitador con motivo obligatorio

### 8.2 Facturacion — Coste interno

| Concepto | Autonomo (actual) | Plantilla (nuevo) |
|----------|-------------------|-------------------|
| A la aseguradora | Factura por baremo (sin cambio) | Factura por baremo (mismo precio) |
| Coste para CuidaCasa | Factura del operario | Coste interno calculado |
| Margen | Baremo cia - Factura operario | Baremo cia - Coste interno |
| Documento | Factura de operario | Parte de trabajo valorizado |

Nuevo sub-apartado en Facturacion: "Costes internos plantilla".

### 8.3 Configuracion — Nuevos apartados

| Apartado | Contenido |
|----------|-----------|
| Reglas del Orquestador | Reglas RO-01 a RO-08: umbrales, prioridades, activacion por zona |
| Baremos de plantilla | Coste/hora por categoria profesional |
| Tipos de turno | Definicion de turnos por delegacion |
| Modelos laborales | Que modelo aplica en cada delegacion |
| Alertas | Umbrales y destinatarios |

### 8.4 Planificacion — Vista unificada

La vista de calendario muestra empleados y autonomos en la misma vista, con codigo de colores diferenciado. Huecos de empleados plantilla resaltados (capacidad infrautilizada).

### 8.5 Informes — Nuevos reportes

| Informe | Periodicidad |
|---------|-------------|
| Rentabilidad plantilla vs autonomo | Mensual |
| Ocupacion de plantilla | Semanal |
| Costes internos | Mensual |
| Decisiones del Orquestador | Mensual |
| Simulacion contratacion | Bajo demanda |

---

## 9. Modelo de Datos

### 9.1 Tablas nuevas

```sql
-- Empleado en plantilla
empleados_plantilla (
    id                  UUID PK,
    nombre_completo     VARCHAR(200) NOT NULL,
    dni                 VARCHAR(20) NOT NULL ENCRYPTED,
    num_seguridad_social VARCHAR(20) NOT NULL ENCRYPTED,
    categoria_profesional ENUM('oficial_1a','oficial_2a','ayudante','jefe_equipo'),
    modelo_laboral      ENUM('multidisciplinar','especializado','equipo'),
    delegacion_id       VARCHAR(10) NOT NULL,
    telefono_1          VARCHAR(20),
    telefono_2          VARCHAR(20),
    email               VARCHAR(200),
    direccion           VARCHAR(300),
    codigo_postal       VARCHAR(10),
    ciudad              VARCHAR(100),
    provincia           VARCHAR(100),
    fecha_alta_empresa  DATE NOT NULL,
    activo              BOOLEAN DEFAULT true,
    foto_url            VARCHAR(500),
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
)

-- Contrato laboral
contratos_laborales (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    tipo_contrato       ENUM('indefinido','temporal','obra_servicio','formacion'),
    jornada             ENUM('completa','parcial'),
    horas_semanales     DECIMAL(4,1) NOT NULL,
    salario_bruto_anual DECIMAL(10,2) NOT NULL,
    convenio            VARCHAR(200),
    fecha_inicio        DATE NOT NULL,
    fecha_fin           DATE,
    activo              BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT NOW()
)

-- Especialidades del empleado
empleado_especialidades (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    especialidad_id     UUID FK especialidades,
    nivel               ENUM('principal','secundario'),
    UNIQUE(empleado_id, especialidad_id)
)

-- Zonas de cobertura del empleado
empleado_zonas (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    zona_id             UUID FK zonas,
    UNIQUE(empleado_id, zona_id)
)

-- Fichajes diarios
fichajes (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    fecha               DATE NOT NULL,
    hora_entrada        TIME NOT NULL,
    hora_salida         TIME,
    horas_totales       DECIMAL(4,2),
    tipo_jornada        ENUM('normal','horas_extra','guardia','festivo'),
    metodo_fichaje      ENUM('app_movil','erp_web','tablet_delegacion'),
    ubicacion_gps       VARCHAR(50),
    created_at          TIMESTAMP DEFAULT NOW(),
    UNIQUE(empleado_id, fecha)
)

-- Turnos
turnos (
    id                  UUID PK,
    nombre              VARCHAR(50) NOT NULL,
    hora_inicio         TIME NOT NULL,
    hora_fin            TIME NOT NULL,
    delegacion_id       VARCHAR(10),
    activo              BOOLEAN DEFAULT true
)

-- Cuadrante mensual
cuadrantes (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    fecha               DATE NOT NULL,
    turno_id            UUID FK turnos,
    tipo_dia            ENUM('trabajo','vacaciones','baja','festivo','formacion','libre'),
    notas               TEXT,
    UNIQUE(empleado_id, fecha)
)

-- Cuadrillas
cuadrillas (
    id                  UUID PK,
    nombre              VARCHAR(100) NOT NULL,
    jefe_equipo_id      UUID FK empleados_plantilla,
    delegacion_id       VARCHAR(10) NOT NULL,
    activa              BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT NOW()
)

-- Miembros de cuadrilla
cuadrilla_miembros (
    id                  UUID PK,
    cuadrilla_id        UUID FK cuadrillas,
    empleado_id         UUID FK empleados_plantilla,
    rol                 ENUM('jefe','operario'),
    UNIQUE(cuadrilla_id, empleado_id)
)

-- Partes de trabajo
partes_trabajo (
    id                  UUID PK,
    expediente_id       UUID FK expedientes,
    servicio_id         UUID,
    empleado_id         UUID FK empleados_plantilla,
    cuadrilla_id        UUID FK cuadrillas,
    fecha               DATE NOT NULL,
    hora_inicio         TIME NOT NULL,
    hora_fin            TIME,
    horas_efectivas     DECIMAL(4,2),
    desplazamiento_km   DECIMAL(6,1),
    estado_trabajo      ENUM('completado','parcial','requiere_revisita','materiales_pendientes'),
    coste_horas         DECIMAL(10,2),
    coste_materiales    DECIMAL(10,2),
    coste_desplazamiento DECIMAL(10,2),
    coste_total         DECIMAL(10,2),
    observaciones       TEXT,
    firma_asegurado_url VARCHAR(500),
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
)

-- Materiales usados en parte de trabajo
parte_materiales (
    id                  UUID PK,
    parte_trabajo_id    UUID FK partes_trabajo,
    nombre_material     VARCHAR(200) NOT NULL,
    cantidad            DECIMAL(8,2) NOT NULL,
    coste_unitario      DECIMAL(10,2) NOT NULL,
    coste_total         DECIMAL(10,2) NOT NULL
)

-- Equipamiento asignado
equipamiento_asignado (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    tipo                ENUM('herramienta','epi','vehiculo'),
    nombre              VARCHAR(200) NOT NULL,
    referencia          VARCHAR(100),
    fecha_entrega       DATE NOT NULL,
    fecha_devolucion    DATE,
    estado              ENUM('activo','reparacion','devuelto','perdido'),
    coste_adquisicion   DECIMAL(10,2),
    notas               TEXT
)

-- Coste interno calculado (snapshot mensual)
costes_internos (
    id                  UUID PK,
    empleado_id         UUID FK empleados_plantilla,
    mes                 DATE NOT NULL,
    salario_bruto_mes   DECIMAL(10,2),
    coste_ss            DECIMAL(10,2),
    prorrata_equipamiento DECIMAL(10,2),
    prorrata_vehiculo   DECIMAL(10,2),
    prorrata_epi        DECIMAL(10,2),
    coste_total_mes     DECIMAL(10,2),
    horas_contratadas   DECIMAL(6,2),
    coste_hora          DECIMAL(8,2),
    UNIQUE(empleado_id, mes)
)

-- Decisiones del Orquestador
decisiones_orquestador (
    id                  UUID PK,
    expediente_id       UUID FK expedientes,
    servicio_id         UUID,
    gremio              VARCHAR(50) NOT NULL,
    zona                VARCHAR(50) NOT NULL,
    fecha_hora          TIMESTAMP DEFAULT NOW(),
    tipo_decision       ENUM('plantilla','autonomo','mixto'),
    regla_aplicada      VARCHAR(10),
    razon               TEXT NOT NULL,
    candidatos_plantilla JSONB,
    empleado_asignado_id UUID FK empleados_plantilla,
    operario_asignado_id UUID,
    coste_plantilla_est DECIMAL(10,2),
    coste_autonomo_est  DECIMAL(10,2),
    delta_margen        DECIMAL(10,2),
    ocupacion_zona      DECIMAL(5,2),
    override_manual     BOOLEAN DEFAULT false,
    override_motivo     TEXT,
    override_usuario_id UUID
)
```

### 9.2 Columnas nuevas en tablas existentes

```sql
ALTER TABLE expediente_servicios ADD COLUMN empleado_id UUID FK empleados_plantilla;
ALTER TABLE expediente_servicios ADD COLUMN tipo_recurso ENUM('plantilla','autonomo') DEFAULT 'autonomo';
ALTER TABLE expediente_servicios ADD COLUMN decision_orquestador_id UUID FK decisiones_orquestador;

ALTER TABLE facturas_compania ADD COLUMN coste_interno DECIMAL(10,2);
ALTER TABLE facturas_compania ADD COLUMN margen_real DECIMAL(10,2);
ALTER TABLE facturas_compania ADD COLUMN tipo_recurso ENUM('plantilla','autonomo');
```

### 9.3 Indices

```sql
CREATE INDEX ix_fichajes_empleado_fecha ON fichajes(empleado_id, fecha);
CREATE INDEX ix_cuadrantes_empleado_fecha ON cuadrantes(empleado_id, fecha);
CREATE INDEX ix_partes_trabajo_expediente ON partes_trabajo(expediente_id);
CREATE INDEX ix_partes_trabajo_empleado_fecha ON partes_trabajo(empleado_id, fecha);
CREATE INDEX ix_decisiones_expediente ON decisiones_orquestador(expediente_id);
CREATE INDEX ix_decisiones_tipo_fecha ON decisiones_orquestador(tipo_decision, fecha_hora);
CREATE INDEX ix_costes_empleado_mes ON costes_internos(empleado_id, mes);
```

---

## 10. Lo que NO incluye esta version

- Nominas (las lleva asesoria externa)
- Vacaciones como workflow de solicitud/aprobacion (se registran en cuadrante directamente)
- PRL / formacion (futuras versiones)
- App movil nativa (fichajes y partes desde ERP web responsive o tablet)
- Integracion con Red guAI (motores separados)
