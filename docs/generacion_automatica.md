# Generación automática de horarios

Cómo el backend produce el calendario mensual de turnos a partir de las reglas almacenadas en BD.

Código relevante: [backend/src/services/generator/](../backend/src/services/generator/).

## 1. Entidades en juego

| Tabla | Qué aporta a la generación |
|---|---|
| `department_members` | Quiénes trabajan · `weekly_hour_limit` por persona · `group_name` (RX / CCEE PLT 0 / …) |
| `shift_types` | Códigos de turno (`M`, `T`, `N`, `D`, `V`, `L`) con horario y flag `counts_as_work_time` |
| `schedule_periods` | Mes a generar (`start_date` → `end_date`) |
| `schedule_assignments` | Asignaciones ya existentes (respetadas si están `is_locked`) |
| `generation_preferences` | Reglas **globales** (singleton) · `general_weekly_hour_limit` + JSONB |
| `member_generation_preferences` | Reglas **por persona** · JSONB con shape descrito abajo |
| `generation_runs` | Histórico de ejecuciones (estrategia usada, resumen) |

## 2. Reglas globales (singleton `generation_preferences`)

En `preferences_jsonb`:

```json
{
  "min_rest_hours": 12,
  "max_consecutive_days": 6,
  "allow_weekend_work": true,
  "prefer_balanced_distribution": true,
  "fill_unassigned_only": true,
  "shift_coverage": {
    "<shift_type_id>": {"min": 2, "max": 5}
  }
}
```

- `min_rest_hours`: horas mínimas entre fin de turno anterior y comienzo del siguiente. Impide, p.ej., T (15–23) seguido de M (07–15) al día siguiente = solo 8h de descanso.
- `max_consecutive_days`: nadie trabaja más de N días seguidos.
- `shift_coverage`: mínimo y máximo de personas por turno y día. Si no hay entrada para un `shift_type_id`, no hay restricción.

## 3. Reglas por persona (`member_generation_preferences.preferences_jsonb`)

Cada persona puede tener hasta 4 campos:

```json
{
  "shift_rotation": ["M", "T"],
  "work_days": [0, 1, 2, 3, 4, 5],
  "daily_hours": 7,
  "weekly_pattern": ["M", "T", "T", "T", "T", "D", "D"]
}
```

| Campo | Tipo | Significado |
|---|---|---|
| `shift_rotation` | array de códigos | Qué turnos puede hacer. Ausente/null = todos. |
| `work_days` | array 0–6 | Días elegibles (0=Lunes, 6=Domingo). Ausente = todos. |
| `daily_hours` | número | Horas contratadas por día. Se usa para contar hacia el límite semanal aunque el turno físico dure más (p.ej. una reducción 5h asignada al turno M de 8h cuenta 5h hacia su semana). |
| `weekly_pattern` | 7 elementos | Turno fijo para cada día de la semana (Lunes→Domingo). Si está, **manda sobre todo lo demás** para esa persona. Elementos pueden ser `null` (el engine decide ese día). |

Las reglas se **inyectan desde el Excel de personal** con [`scripts/import_personal_mostrador.py`](../backend/scripts/import_personal_mostrador.py) — ver §8.

## 4. El pipeline de generación

Orquestado en [`engine.py`](../backend/src/services/generator/engine.py) `run_generation()`:

1. **Carga** el período, los miembros activos, sus preferencias por persona, los shift types activos, las asignaciones existentes y las preferencias globales.
2. Construye un `GenerationContext` inmutable con toda esa información + el rango de fechas.
3. Selecciona la estrategia (`balanced`, `coverage`, `conservative`) del diccionario `STRATEGIES`.
4. La estrategia devuelve una lista de `ProposedAssignment` (member_id, date, shift_type_id).
5. Cada propuesta se inserta vía `assignment_service.create_assignment()` con `assignment_source="auto"`.
6. Se guarda un `GenerationRun` con el resumen (input_snapshot + result_summary) para auditoría.

## 5. Las tres estrategias

Todas recorren `for fecha in dates: for miembro in sorted_por_horas: decidir_turno`. Se diferencian en cómo desempatan y qué tan flexibles son con los límites.

### `balanced` — equilibrio y cobertura
- Ordena miembros por horas trabajadas (los menos cargados primero) y los distribuye.
- Respeta cobertura mín/máx por turno por día.
- Cuando la cobertura está corta, prioriza ese turno; cuando está sobrada, lo excluye.
- Es la estrategia por defecto y la más sensata para horarios nuevos.

### `coverage` — prioriza llenar huecos
- Permite hasta 110% del límite semanal por persona si hace falta para dar cobertura.
- Útil si un mes hay poca gente disponible.

### `conservative` — solo rellena vacíos
- Respeta asignaciones existentes sin intentar optimizar.
- Ideal tras editar manualmente el horario y querer completar solo los días vacíos.

## 6. El camino corto: `weekly_pattern`

Si una persona tiene `weekly_pattern` definido, **las tres estrategias lo aplican tal cual** al principio del bucle y saltan el resto de la lógica para esa persona ese día:

```
código[weekday(d)] → pone ese turno, suma horas si corresponde, siguiente
```

Esto significa que para miembros con patrón:
- **No** se comprueba cobertura por turno.
- **No** se comprueba días consecutivos.
- **No** se comprueba descanso min_rest_hours entre turnos.
- **Sí** se suma `daily_hours` al acumulador semanal (por si se mezcla con otros días fuera de patrón).

El patrón es autoridad: si tú como jefa diseñas `[M, T, T, T, T, D, D]`, el generador lo pone cada semana sin preguntar. Las reglas `shift_rotation` / `work_days` / `daily_hours` siguen guardadas pero no se consultan cuando hay patrón.

## 7. Cómputo de horas: `hours_for(shift)`

Definido en [`MemberInfo`](../backend/src/services/generator/base.py):

```python
def hours_for(self, shift):
    if self.daily_hours is not None and shift.counts_as_work_time:
        return min(shift.hours, self.daily_hours)
    return shift.hours
```

Permite que una persona de reducción 5h asignada al turno M (8h físicas) cuente solo 5h hacia su `weekly_hour_limit`. La asignación en BD sigue siendo "M" — el HR/export se encarga de reflejar que se va a las 12:00 en lugar de a las 15:00.

Si en el futuro se quiere **turno físico corto** (un M7 que termine a las 14:00), crear shift types adicionales (`M7`, `M5`, `T7`, `T5`) y referenciarlos desde el `weekly_pattern` de cada persona.

## 8. Scripts útiles

### Sembrar miembros desde el Excel de personal

```bash
cd backend
uv run python -m scripts.import_personal_mostrador [ruta/al/xlsx]
```

Lee un Excel tipo [`PERSONAL MOSTRADOR.xlsx`](../backend/PERSONAL%20MOSTRADOR.xlsx) con columnas `nombre | jornada | turnos | días`, detecta los grupos por cabecera de fila (`PERSONAL MOSTRADOR RX`, `MOSTRADOR LABORATORIO`, etc.), y crea/actualiza:

- `DepartmentMember` con `group_name`, `weekly_hour_limit`, `role_name`.
- `MemberGenerationPreference` con `shift_rotation`, `work_days`, `daily_hours`.

Idempotente por `full_name`.

### Aprender patrones semanales de un mes histórico

```bash
cd backend
uv run python -m scripts.learn_patterns_from_xlsx [ruta/al/xlsx]
```

Dado un Excel con formato de matriz (fila = miembro, columnas = 30 días etiquetados `Lun\n1` … `Vie\n30`), extrae el turno **más frecuente** para cada día de la semana de cada persona y lo guarda como `weekly_pattern` en su JSONB.

Con esto, tu horario histórico se convierte en regla: la próxima generación producirá exactamente la misma distribución semanal.

## 9. Añadir o cambiar reglas a mano

Por ahora, la UI no expone todos los campos JSONB. Para editar:

```sql
UPDATE member_generation_preferences
SET preferences_jsonb = preferences_jsonb ||
    '{"weekly_pattern": ["M","T","T","T","T","D","D"]}'::jsonb
WHERE member_id = (
    SELECT id FROM department_members WHERE full_name = 'ANA CALVO RAMOS'
);
```

Reglas frecuentes de tocar:
- `weekly_pattern[5]` (Sábado) para que alguien cubra fines de semana.
- `work_days` para incluir/excluir días elegibles.
- `shift_rotation` para restringir a solo M o solo T.

## 10. Qué no hay (todavía)

Limitaciones conocidas del motor actual:

- **Rotación par/impar semana**: no hay un `weekly_pattern_alt`. Si quieres que CHABELI haga M una semana y T a la siguiente, hoy no se puede — su patrón es fijo.
- **Cobertura por día de la semana**: `shift_coverage` es una única pareja min/max por turno para **todos los días**. No se puede decir "los lunes necesito más M que los jueves".
- **Preferencias suaves**: el generador no tiene concepto de "prefiero tarde pero puedo mañana si hace falta". Es binario (permitido / no permitido).
- **Bloqueo de fechas concretas** (vacaciones, bajas): se modela como asignación manual previa con `is_locked=true`. No hay un campo "de X a Y no está disponible".

Si alguna de estas se vuelve necesaria, el sitio natural es ampliar el JSONB `preferences_jsonb` y la clase `MemberInfo` — no hace falta tocar schema relacional.
