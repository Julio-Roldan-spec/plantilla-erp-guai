# Modulo Plantilla — ERP guAI

**Gestion de Operarios en Plantilla + Orquestador de Asignacion Hibrida**

Ampliacion del ERP operativo guAI de Grupo CuidaCasa para gestionar operarios contratados en plantilla junto al modelo actual de operarios autonomos, con un motor inteligente que optimiza la asignacion por coste, disponibilidad y zona.

---

## Problema que resuelve

El ERP guAI gestiona siniestros de hogar (multiasistencia) para companias aseguradoras. Actualmente, todos los operarios son autonomos externos. Este modulo permite:

- **Gestionar empleados propios** con control de jornada, partes de trabajo y costes internos
- **Decidir automaticamente** cuando asignar plantilla vs autonomo en cada servicio
- **Medir la rentabilidad real** de tener plantilla propia vs subcontratar autonomos
- **Simular contrataciones** para saber donde es rentable incorporar empleados

---

## Arquitectura

```
Nuevo servicio en expediente
        |
        v
   +-------------------+
   |   ORQUESTADOR      |  Reglas configurables: coste, zona, jornada, ocupacion
   |   de Asignacion    |
   +--------+-----------+
            |
     +------+------+
     |             |
     v             v
  Plantilla     Red guAI
  (empleados)   (autonomos)
     |             |
     v             v
  Parte de      Factura
  Trabajo       operario
     |             |
     v             v
   +-----------------+
   | Dashboard de    |  Ahorro, ocupacion, simulador, alertas
   | Rentabilidad    |
   +-----------------+
```

**Orquestador en 4 fases:**

1. **Filtrado**: Hay empleado disponible? (zona, gremio, jornada, ausencias)
2. **Coste**: Es mas rentable plantilla o autonomo? (delta margen)
3. **Decision**: 8 reglas configurables por delegacion (RO-01 a RO-08)
4. **Registro**: Trazabilidad completa de cada decision

---

## Modulos

### Plantilla (nuevo en sidebar)

| Seccion | Descripcion |
|---------|-------------|
| **Empleados** | Ficha laboral completa con 10 pestanas |
| **Partes de Trabajo** | Horas + materiales + firma por expediente/servicio |
| **Jornada y Fichajes** | Control horario legal (RD 8/2019) |
| **Turnos y Cuadrantes** | Planificacion semanal/mensual |
| **Equipamiento** | Herramientas, EPIs, vehiculos asignados |
| **Dashboard Rentabilidad** | KPIs, graficos, simulador, alertas |

### Integracion con ERP existente

| Modulo | Cambio |
|--------|--------|
| **Expedientes** | Asignacion dual: empleado o autonomo por servicio |
| **Facturacion** | Nuevo concepto Coste interno + margen real |
| **Planificacion** | Vista unificada empleados + autonomos |
| **Configuracion** | Reglas del Orquestador, baremos plantilla, turnos |
| **Informes** | 5 reportes nuevos de rentabilidad |

---

## Ficha del Empleado

**Datos generales:** Nombre, DNI (cifrado), N.SS (cifrado), categoria profesional, delegacion, contacto, direccion.

**10 pestanas:**

| Pestana | Contenido |
|---------|-----------|
| Contrato | Tipo, jornada, horas semanales, salario, convenio |
| Especialidades | Gremios (principal/secundario) |
| Zonas | Zonas de cobertura asignadas |
| Jornada | Fichajes del mes, horas vs contratadas, extras |
| Partes de Trabajo | Lista por expediente con horas, materiales, coste |
| Turnos | Cuadrante mensual con turno por dia |
| Equipamiento | Herramientas, EPIs, vehiculo asignado |
| Cuadrilla | Miembros del equipo (modelo equipos) |
| Coste | Desglose coste/hora: salario + SS + prorratas |
| Calendario | Vacaciones, bajas, festivos, formacion |

**3 modelos laborales** (configurables por delegacion):
- **Multidisciplinar**: Un empleado cubre varios gremios
- **Especializado**: 1-2 gremios fijos por empleado
- **Equipos**: Cuadrillas con jefe de equipo

---

## Orquestador — Reglas de Decision

| Regla | Condicion | Accion |
|-------|-----------|--------|
| RO-01 | Margen plantilla > margen autonomo | Asignar **plantilla** |
| RO-02 | Margen ligeramente inferior pero ocupacion < 70% | Asignar **plantilla** (aprovechar capacidad) |
| RO-03 | Margen muy inferior | Derivar a **Red guAI** (autonomo) |
| RO-04 | Urgencia fuera de horario laboral | Derivar a **Red guAI** |
| RO-05 | Empleado a >45 min, autonomo a <20 min | Derivar a **Red guAI** |
| RO-06 | Requiere cuadrilla, solo 1 empleado | **Red guAI** o **Mixto** |
| RO-07 | Ocupacion zona > 90% | **Red guAI** + alerta saturacion |
| RO-08 | Ocupacion zona < 40% | Forzar **plantilla** (evitar infrautilizacion) |

Todas las decisiones generan registro trazable con: candidatos evaluados, regla aplicada, costes estimados, delta margen.

El tramitador puede hacer **override manual** con motivo obligatorio.

---

## Dashboard de Rentabilidad

### KPIs

| KPI | Descripcion |
|-----|-------------|
| Ahorro mensual | Suma de delta margen positivo del mes |
| Ocupacion plantilla | Horas productivas / horas contratadas (objetivo: 70-85%) |
| Coste medio/servicio | Total coste interno / servicios completados |
| % cobertura plantilla | Servicios plantilla / total servicios |

### Simulador de Contratacion

Responde: Si contrato X personas en zona Y, cuanto ahorro?

- Input: zona, gremio, n. empleados, jornada, salario estimado
- Output: expedientes capturados, ahorro mensual, coste empleados, punto equilibrio, ROI anualizado, recomendacion

### Alertas Proactivas

| Alerta | Trigger |
|--------|---------|
| Saturacion | Ocupacion > 90% durante 5 dias |
| Infrautilizacion | Ocupacion < 40% durante 10 dias |
| Oportunidad contratacion | >20 servicios/mes sin plantilla con margen positivo |
| Vencimiento contrato | Temporal a 30 dias de fin |
| Desequilibrio gremio | >80% autonomo en zona con plantilla |
| Horas extra excesivas | >20h extra/mes durante 2 meses |

---

## Ontologia

### Entidades nuevas (10)

| Entidad | Descripcion |
|---------|-------------|
| Empleado | Operario contratado (datos laborales) |
| ContratoLaboral | Tipo, jornada, salario, vigencia |
| ParteTrabajo | Horas + materiales por expediente |
| Fichaje | Entrada/salida diaria |
| Turno | Definicion horaria (manana/tarde/partido/guardia) |
| Cuadrante | Planificacion de turnos por dia |
| Cuadrilla | Grupo con jefe de equipo |
| EquipamientoAsignado | Herramienta/EPI/vehiculo |
| CosteInterno | Coste/hora calculado mensual |
| DecisionOrquestador | Registro trazable de cada decision |

### Reglas inmutables (12)

| Regla | Descripcion |
|-------|-------------|
| R-PLA-01 | No asignar fuera de jornada sin registro horas extra |
| R-PLA-02 | No superar 10h diarias (limite legal) |
| R-PLA-03 | No asignar en vacaciones, baja o ausencia |
| R-PLA-04 | No asignar a gremio no registrado en ficha |
| R-PLA-05 | No asignar fuera de zonas asignadas |
| R-PLA-06 | Todo servicio plantilla genera ParteTrabajo |
| R-PLA-07 | Toda decision genera DecisionOrquestador |
| R-PLA-08 | Sin fichaje del dia = sin asignaciones |
| R-PLA-09 | Coste interno recalculado mensualmente |
| R-PLA-10 | Cuadrilla se asigna completa, no individual |
| R-PLA-11 | Descanso minimo 12h entre jornadas |
| R-PLA-12 | Alerta 30 dias antes de vencimiento temporal |

---

## Modelo de Datos

### Tablas nuevas (13)

```
empleados_plantilla          Ficha del empleado
contratos_laborales          Contrato laboral activo + historico
empleado_especialidades      N:M con tabla especialidades existente
empleado_zonas               N:M con tabla zonas existente
fichajes                     Registro diario entrada/salida
turnos                       Definicion de tipos de turno
cuadrantes                   Turno asignado por dia por empleado
cuadrillas                   Grupos de trabajo
cuadrilla_miembros           Miembros de cada cuadrilla
partes_trabajo               Registro por servicio/expediente
parte_materiales             Materiales usados en cada parte
equipamiento_asignado        Herramientas/EPIs/vehiculos
costes_internos              Snapshot mensual coste/hora
decisiones_orquestador       Trazabilidad de cada decision
```

### Columnas nuevas en tablas existentes

```
expediente_servicios  + empleado_id, tipo_recurso, decision_orquestador_id
facturas_compania     + coste_interno, margen_real, tipo_recurso
```

---

## Cobertura Inicial

| Delegacion | Provincia | Estado |
|------------|-----------|--------|
| ALC | Alicante | Fase 1 |
| VLC | Valencia | Fase 1 |
| CAS | Castellon | Fase 1 |

Expansion a otras zonas configurable cuando el simulador indique rentabilidad.

---

## Lo que NO incluye esta version

- Nominas (asesoria externa)
- Workflow de solicitud/aprobacion de vacaciones (registro directo en cuadrante)
- PRL / formacion (futuras versiones)
- App movil nativa (ERP web responsive + tablet)
- Integracion con Red guAI (motores separados)

---

## Stack Tecnologico

Mismo stack que el ERP guAI existente:

- **Backend**: Python + FastAPI
- **Base de datos**: PostgreSQL
- **Frontend**: Vue 3 + TypeScript + Tailwind CSS
- **Estado**: Pinia
- **Tiempo real**: WebSocket
- **Contenedores**: Docker

---

## Documentacion

| Documento | Ubicacion |
|-----------|-----------|
| Spec de diseno completa | [docs/specs/design.md](docs/specs/design.md) |

---

## Licencia

Proyecto privado de Grupo CuidaCasa / guAI Insurtech.
