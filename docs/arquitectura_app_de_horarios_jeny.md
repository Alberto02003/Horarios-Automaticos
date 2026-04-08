# 📅 Aplicación de Gestión de Horarios - Jeny

## 🧱 Stack Tecnológico

### Frontend
- React + TypeScript
- Vite (build y desarrollo)
- TanStack Query (gestión de estado servidor)
- Zustand (estado local)
- Tailwind CSS (UI)
- FullCalendar (visualización de calendario)

### Multiplataforma
- Tauri 2
  - Instalador escritorio (Windows, macOS, Linux)
  - APK Android (tablet)

### Backend
- Python 3.12
- FastAPI
- SQLAlchemy 2.x
- Pydantic v2
- Alembic (migraciones)
- psycopg 3 (driver PostgreSQL)

### Base de Datos
- PostgreSQL
- Uso de modelo híbrido:
  - Relacional (estructura principal)
  - JSONB (configuración flexible y metadatos)

### Infraestructura
- Docker
- Nginx (reverse proxy)
- Backups automáticos de base de datos

---

# 🧩 Modelo de Datos

## 1. users
Autenticación de usuarios (aunque inicialmente solo Jeny)

Campos:
- id
- email
- password_hash
- display_name
- is_active
- created_at
- updated_at

---

## 2. department_members
Personas del departamento

Campos:
- id
- full_name
- role_name
- weekly_hour_limit
- is_active
- color_tag
- metadata_jsonb

---

## 3. schedule_periods
Representa un periodo (normalmente mensual)

Campos:
- id
- name
- year
- month
- start_date
- end_date
- status (draft | active)
- generation_summary_jsonb
- created_by_user_id
- activated_at
- created_at
- updated_at

---

## 4. shift_types
Tipos de turno

Campos:
- id
- code
- name
- category (work | vacation | special)
- default_start_time
- default_end_time
- counts_as_work_time
- color
- is_active

---

## 5. schedule_assignments
Asignaciones por persona y día

Campos:
- id
- schedule_period_id
- member_id
- date
- shift_type_id
- start_time
- end_time
- assignment_source (manual | auto | template)
- is_locked
- details_jsonb
- created_at
- updated_at

---

## 6. generation_preferences
Configuración global del generador

Campos:
- id
- general_weekly_hour_limit
- preferences_jsonb
- updated_at

Ejemplo JSON:
```
{
  "min_rest_hours": 12,
  "max_consecutive_days": 6,
  "allow_weekend_work": true,
  "prefer_balanced_distribution": true,
  "fill_unassigned_only": true
}
```

---

## 7. member_generation_preferences
Preferencias por integrante

Campos:
- id
- member_id
- preferences_jsonb
- updated_at

---

## 8. generation_runs
Historial de ejecuciones automáticas

Campos:
- id
- schedule_period_id
- executed_by_user_id
- strategy
- input_snapshot_jsonb
- result_summary_jsonb
- created_at

---

# ⚙️ Funcionalidades

## 🔐 Autenticación
- Login simple
- Usuario único inicialmente

## 👥 Gestión de equipo
- Crear/editar/eliminar integrantes
- Definir roles
- Definir límite de horas

## ⚙️ Configuración
- Preferencias globales de generación
- Preferencias por integrante

## 📆 Planificación
- Vista mensual
- Vista semanal
- Edición manual rápida
- Asignación de turnos
- Marcar vacaciones

## 🤖 Generación automática
- Generar horario completo
- Generar solo huecos
- Regeneración parcial
- Estrategias:
  - Equilibrado
  - Cobertura
  - Conservador

## 🔒 Control manual
- Bloqueo de asignaciones
- Edición posterior a generación

## 📊 Validaciones
- Control de horas
- Detección de conflictos
- Avisos de sobrecarga

## ✅ Flujo de estados
- Borrador
- Activo (aprobado por Jeny)

## 📤 Exportación
- Exportación a Excel
- Exportación a PDF

## 🧠 UX Avanzada
- Colores por tipo de turno
- Indicadores por persona (horas, diferencias)
- Panel de conflictos
- Edición masiva

## 🧾 Historial
- Registro de generaciones
- Métricas de generación

---

# 🧠 Concepto clave del sistema

El sistema se basa en un modelo de:

> Generación automática + revisión manual + aprobación final

Donde:
- La app propone
- Jeny revisa
- Jeny decide

Esto garantiza control total con automatización asistida.

---

# 🚀 Futuras mejoras

- Plantillas de horarios
- IA para optimización
- Multiusuario
- Notificaciones
- Integración con calendarios externos

---

# 📌 Conclusión

El sistema está diseñado para ser:
- Escalable
- Flexible
- Fácil de usar
- Preparado para automatización avanzada

Sin perder simplicidad en el uso diario.

