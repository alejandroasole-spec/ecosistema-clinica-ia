# Esquema de la base Airtable — "Ecosistema Clinica"

Base con 4 tablas vinculadas. El orden de creación importa: creá primero
`Prioridades` y `Pacientes`, porque `Solicitudes` las referencia.

---

## Tabla 1 — Prioridades  (catálogo maestro)

Tabla de referencia que evita hardcodear los niveles en el flujo.

| Campo | Tipo | Valores / notas |
|---|---|---|
| Nivel | Single line text (campo primario) | ALTA · NORMAL |
| SLA_minutos | Number (integer) | ALTA = 15 · NORMAL = 1440 |
| Requiere_aprobacion | Checkbox | ALTA = true · NORMAL = false |
| Canal_salida | Single select | Slack · Gmail |
| Descripcion | Long text | Criterio de negocio del nivel |

**Registros iniciales (cargar a mano, 2 filas):**

| Nivel | SLA_minutos | Requiere_aprobacion | Canal_salida | Descripcion |
|---|---|---|---|---|
| ALTA | 15 | ✅ | Slack | Urgencia médica, dolor agudo, sangrado, cirugía inminente o reclamo grave. |
| NORMAL | 1440 | ☐ | Gmail | Consultas administrativas, turnos, facturación e información general. |

---

## Tabla 2 — Pacientes  (memoria del sistema)

Da continuidad entre solicitudes: el sistema recuerda a quién ya atendió.

| Campo | Tipo | Notas |
|---|---|---|
| Email | Email (campo primario) | Clave de deduplicación |
| Nombre | Single line text | Puede quedar vacío |
| Total_solicitudes | Count → Solicitudes | Se calcula solo |
| Ultima_interaccion | Date | Actualizado por el flujo |
| Solicitudes | Link to Solicitudes | Relación inversa |
| Notas_internas | Long text | Contexto que carga el equipo |

---

## Tabla 3 — Solicitudes  (tabla operativa principal)

Una fila por correo entrante. Es el centro de comando del sistema.

| Campo | Tipo | Escribe | Notas |
|---|---|---|---|
| ID_Solicitud | Autonumber (campo primario) | Airtable | |
| Message_ID | Single line text | Make | ID de Gmail. Evita duplicados. |
| Thread_ID | Single line text | Make | Mantiene el hilo de conversación. |
| Asunto | Single line text | Make | |
| Cuerpo | Long text | Make | Texto del correo. |
| Remitente | Email | Make | |
| Paciente | Link to Pacientes | Make | Relación por email. |
| **Estado** | **Single select** | **Make + Humano** | **Pendiente · Procesado por IA · Esperando aprobación · Aprobado por Humano · Enviado · Rechazado · Error** |
| Prioridad_IA | Link to Prioridades | Make | Resultado del nodo de IA. |
| Justificacion_IA | Long text | Make | Por qué la IA clasificó así. |
| Confianza_IA | Number (decimal, 2) | Make | 0.00 a 1.00 |
| Borrador_Respuesta | Long text | Make | Texto propuesto al paciente. |
| **Aprobado** | **Checkbox** | **Solo Humano** | **El semáforo HITL. Fuera de permisos de Make.** |
| Revisor | Single line text | Humano | Quién aprobó. |
| Fecha_aprobacion | Date (con hora) | Humano | |
| Fecha_ingreso | Created time | Airtable | |
| Tiempo_respuesta_min | Formula | Airtable | `DATETIME_DIFF(Fecha_aprobacion, Fecha_ingreso, 'minutes')` |
| Dentro_SLA | Formula | Airtable | `IF(Tiempo_respuesta_min <= SLA_minutos, "SI", "NO")` |
| Canal_enviado | Single select | Make | Slack · Gmail · Ninguno |
| Errores | Link to Log_Errores | Make | Relación inversa. |

---

## Tabla 4 — Log_Errores  (resiliencia)

Todo fallo queda registrado acá. Es la fuente del KPI de tasa de errores.

| Campo | Tipo | Notas |
|---|---|---|
| ID_Error | Autonumber (campo primario) | |
| Solicitud | Link to Solicitudes | Qué solicitud falló |
| Modulo | Single select | Gmail Trigger · Airtable Upsert · Nodo IA · Slack · Gmail Salida |
| Tipo_error | Single select | API caída · Timeout · Dato faltante · Respuesta inválida · Sin conexión |
| Mensaje | Long text | Texto crudo del error |
| Fecha | Created time | |
| Resuelto | Checkbox | Marcado por el equipo |
| Reintentos | Number (integer) | Cuántas veces reintentó el handler |

---

## Relaciones entre tablas

```
Pacientes  1 ──────< N  Solicitudes  N >────── 1  Prioridades
                              │
                              │ 1
                              │
                              ˅ N
                        Log_Errores
```

- Un paciente tiene muchas solicitudes.
- Cada solicitud apunta a un nivel de prioridad del catálogo.
- Una solicitud puede acumular varios errores.

Ningún dato queda aislado: los niveles de prioridad y los SLA viven en su
propia tabla en lugar de estar escritos dentro del flujo.

---

## Vistas a crear en Solicitudes

| Vista | Filtro | Para qué |
|---|---|---|
| Cola de procesamiento | Estado = Pendiente | Trigger del escenario 1 |
| Esperando aprobación | Estado = "Esperando aprobación" AND Aprobado = false | Bandeja del revisor humano |
| Listas para enviar | Aprobado = true AND Estado = "Aprobado por Humano" | Trigger del escenario 2 |
| **Dashboard KPIs** | sin filtro, agrupada por Estado | Vista pública compartida |

---

## Dashboard de control (entregable 5)

Crear una **Interface** en Airtable (pestaña Interfaces) con:

1. **Contador** — total de solicitudes del mes.
2. **Gráfico de torta** — distribución por Estado.
3. **Gráfico de barras** — solicitudes por Prioridad_IA.
4. **Número grande** — tasa de errores: registros en Log_Errores sobre total de solicitudes.
5. **Número grande** — cumplimiento de SLA: porcentaje con Dentro_SLA = "SI".
6. **Tiempo medio de aprobación** — promedio de Tiempo_respuesta_min.
7. **Tabla** — últimos 10 errores sin resolver.

Publicar con **Share view → Create a shareable link**. Ese enlace es el que va al README.
