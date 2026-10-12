# Modelo de Datos — Certificado de Estudios

## 1. Propósito

Base de datos para emitir un **Certificado de Estudios** universitario idéntico al
documento de ejemplo: encabezado con datos del alumno y la institución, detalle de
TODAS las materias del plan de estudios con sus calificaciones, resumen por curso,
Trabajo de Conclusión de Carrera (TCC), título otorgado, totales, escalas y firmas.

Principio rector: **no se almacena ningún dato derivado**. Promedios, totales y
resúmenes se calculan al momento de generar el documento, por lo que nunca quedan
desactualizados. El único dato propio del certificado es su `fecha_expedicion`.

## 2. Estructura del modelo (14 tablas + 1 enum)

| Grupo | Tablas | Rol |
|---|---|---|
| Estructura académica | `institucion`, `carrera`, `plan_estudio`, `materia` | Qué se estudia: planes con materias organizadas en año/bimestre |
| Personas | `alumno`, `profesor`, `funcionario` | Actores del documento |
| Inscripción | `alumno_carrera` | Qué alumno cursa qué carrera (N:M) |
| Evaluación | `examen`, `calificacion` | Cómo se obtuvo cada nota (acta, fecha, profesor) |
| Catálogos | `escala_calificacion`, `periodo_evaluacion`, `modalidad`, `tipo_materia` (enum) | Valores controlados impresos en "Observaciones" |
| Documento | `certificado`, `certificado_firma` | El certificado emitido y sus firmantes |

### Reglas estructurales clave

- `materia`: 5 bimestres por año, **máximo 2 materias REGULAR por bimestre**.
  `tipo` distingue REGULAR / TCC / PASANTIA.
- `examen`: cada calificación proviene de un examen con **nº de acta único** y
  **profesor asignado**. El acta lleva además fecha, periodo (O/E) y modalidad (P/D).
- `escala_calificacion`: catálogo 0–5 y 5F. De aquí salen la **nota en letras**
  impresa y el `valor_numerico` para promedios (5F = 5.0). Aprobado desde 2.
- `certificado`: FK compuesta uno-a-uno hacia `alumno_carrera` (un certificado
  por alumno por carrera) + FK al `plan_estudio` que se detalla en el documento.

## 3. Cómo funciona (flujo para emitir un certificado)

1. Se carga la estructura: institución → carrera → plan de estudios → materias.
2. Se inscribe al alumno en la carrera (`alumno_carrera`).
3. Se registran exámenes (acta, profesor, fecha, periodo, modalidad) y las
   calificaciones de cada alumno en cada examen.
4. Al emitir: `INSERT INTO certificado (alumno_id, carrera_id, plan_estudio_id,
   fecha_expedicion)` + sus firmas en `certificado_firma`.
5. El documento se genera con consultas (sección 4): las **filas salen del plan
   de estudios** (todas las materias, ordenadas por año y bimestre) y cada fila
   se completa con la nota del alumno vía LEFT JOIN.

```
certificado ─► plan_estudio ─► materia (TODAS las filas impresas)
     │                              └─► examen ─► calificacion ─► escala (nota)
     ├─► alumno_carrera ─► alumno (datos personales)
     ├─► carrera ─► institucion (nombre, sede = lugar_expedicion)
     └─► certificado_firma ─► funcionario (firmas)
```

## 4. Consultas de generación (por sección del documento)

### 4.1 Encabezado

Datos de institución, carrera, alumno y plan: joins directos desde `certificado`.

```sql
SELECT i.nombre AS universidad, i.sede, i.ciudad,
       ca.nombre AS carrera, ca.titulo_otorgado, ca.duracion_anios,
       a.apellidos, a.nombres, a.sexo, a.nacionalidad, a.nro_documento,
       p.descripcion AS plan_estudio, c.fecha_expedicion
FROM   certificado c
JOIN   alumno a        ON a.id = c.alumno_id
JOIN   carrera ca      ON ca.id = c.carrera_id
JOIN   institucion i   ON i.id = ca.institucion_id
JOIN   plan_estudio p  ON p.id = c.plan_estudio_id
WHERE  c.id = :id_certificado;
```

### 4.2 Detalle de materias (filas del certificado)

```sql
SELECT m.anio_curso, m.bimestre, m.nombre       AS materia,
       ec.codigo        AS calificacion_nro,
       ec.nota_en_letras,
       m.horas_catedra  AS carga_horaria,
       e.nro_acta, e.fecha,
       e.periodo_codigo AS periodo_evaluacion,
       e.modalidad_codigo
FROM   certificado c
JOIN   plan_estudio p        ON p.id = c.plan_estudio_id
JOIN   materia m             ON m.plan_estudio_id = p.id
LEFT JOIN examen e           ON e.materia_id = m.id
LEFT JOIN calificacion cal   ON cal.examen_id = e.id
                            AND cal.alumno_id = c.alumno_id
LEFT JOIN escala_calificacion ec ON ec.codigo = cal.escala_codigo
WHERE  c.id = :id_certificado
ORDER  BY m.anio_curso, m.bimestre, m.nombre;
```

El LEFT JOIN garantiza que aparezcan TODAS las materias del plan aunque el alumno
aún no las haya cursado (fila sin nota, como en un certificado en trámite).

### 4.3 Resumen por curso ("PRIMER CURSO ... Curso Completo / Horas / Promedio")

```sql
SELECT m.anio_curso,
       SUM(m.horas_catedra)                 AS horas_catedras,
       TRUNCATE(AVG(ec.valor_numerico), 2)  AS promedio_obtenido,
       (COUNT(*) = SUM(ec.es_aprobatoria))  AS curso_completo
FROM   certificado c
JOIN   plan_estudio p        ON p.id = c.plan_estudio_id
JOIN   materia m             ON m.plan_estudio_id = p.id AND m.tipo = 'REGULAR'
JOIN   examen e              ON e.materia_id = m.id
JOIN   calificacion cal      ON cal.examen_id = e.id AND cal.alumno_id = c.alumno_id
JOIN   escala_calificacion ec ON ec.codigo = cal.escala_codigo
WHERE  c.id = :id_certificado
GROUP  BY m.anio_curso;
```

### 4.4 Bloque TCC

Fila de la materia `tipo = 'TCC'`; el título del trabajo sale de
`calificacion.titulo_trabajo`.

### 4.5 Totales generales

```sql
SELECT TRUNCATE(AVG(ec.valor_numerico), 2) AS promedio_general,
       SUM(CASE WHEN m.tipo = 'REGULAR'  THEN m.horas_catedra ELSE 0 END) AS total_horas_catedras,
       SUM(CASE WHEN m.tipo = 'PASANTIA' THEN m.horas_catedra ELSE 0 END) AS horas_pasantia,
       SUM(CASE WHEN m.tipo = 'TCC'      THEN m.horas_catedra ELSE 0 END) AS horas_tcc
FROM   certificado c
JOIN   plan_estudio p        ON p.id = c.plan_estudio_id
JOIN   materia m             ON m.plan_estudio_id = p.id
JOIN   examen e              ON e.materia_id = m.id
JOIN   calificacion cal      ON cal.examen_id = e.id AND cal.alumno_id = c.alumno_id
JOIN   escala_calificacion ec ON ec.codigo = cal.escala_codigo
WHERE  c.id = :id_certificado;
-- total_general_horas = total_horas_catedras + horas_pasantia + horas_tcc
-- duracion carrera    = carrera.duracion_anios
-- lugar_expedicion    = institucion.sede
```

### 4.6 Observaciones y firmas

- Escalas y códigos: `SELECT * FROM escala_calificacion / periodo_evaluacion / modalidad`.
- Firmas: `funcionario JOIN certificado_firma ORDER BY orden`.

## 5. Triggers y lógica a implementar

### 5.1 Integridad de la estructura del plan

- **CHECK** `materia.bimestre BETWEEN 1 AND 5`; `alumno.sexo IN ('M','F')`.
- **TRIGGER `trg_materia_max2_bimestre`** (BEFORE INSERT/UPDATE ON `materia`):
  rechazar si ya existen 2 materias REGULAR en el mismo
  (plan_estudio_id, anio_curso, bimestre).
- **TRIGGER `trg_materia_unicos_especiales`**: máximo 1 TCC y 1 PASANTIA por plan.
- **TRIGGER `trg_plan_sin_solape`** (BEFORE INSERT ON `plan_estudio`): impedir
  rangos de vigencia superpuestos para la misma carrera.

```sql
DELIMITER //
CREATE TRIGGER trg_materia_max2_bimestre
BEFORE INSERT ON materia
FOR EACH ROW
BEGIN
  DECLARE n INT;
  SELECT COUNT(*) INTO n
    FROM materia
   WHERE plan_estudio_id = NEW.plan_estudio_id
     AND anio_curso      = NEW.anio_curso
     AND bimestre        = NEW.bimestre
     AND tipo            = 'REGULAR';
  IF NEW.tipo = 'REGULAR' AND n >= 2 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Maximo 2 materias REGULAR por bimestre';
  END IF;
END//
DELIMITER ;
```

### 5.2 Integridad de la evaluación

- **TRIGGER `trg_calificacion_inscripcion`** (BEFORE INSERT ON `calificacion`):
  verificar que el alumno esté inscripto (`alumno_carrera`) en la carrera a la que
  pertenece la materia del examen.
- **UNIQUE** `examen.nro_acta` y `calificacion(examen_id, alumno_id)`: un acta no
  se repite y un alumno no tiene dos notas en el mismo examen.

```sql
DELIMITER //
CREATE TRIGGER trg_calificacion_inscripcion
BEFORE INSERT ON calificacion
FOR EACH ROW
BEGIN
  DECLARE ok INT;
  SELECT COUNT(*) INTO ok
    FROM examen e
    JOIN materia m        ON m.id = e.materia_id
    JOIN plan_estudio p   ON p.id = m.plan_estudio_id
    JOIN alumno_carrera ac ON ac.carrera_id = p.carrera_id
                         AND ac.alumno_id  = NEW.alumno_id
   WHERE e.id = NEW.examen_id;
  IF ok = 0 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'El alumno no esta inscripto en la carrera del examen';
  END IF;
END//
DELIMITER ;
```

### 5.3 Emisión del certificado

- **TRIGGER `trg_certificado_plan_valido`** (BEFORE INSERT ON `certificado`):
  1. `plan_estudio.carrera_id` debe ser = `NEW.carrera_id`;
  2. el plan debe estar completo: no debe existir ninguna materia REGULAR/TCC del
     plan sin calificación aprobatoria del alumno. Si falta alguna, se rechaza.

```sql
DELIMITER //
CREATE TRIGGER trg_certificado_plan_valido
BEFORE INSERT ON certificado
FOR EACH ROW
BEGIN
  DECLARE faltantes INT;
  DECLARE carrera_del_plan INT;

  SELECT carrera_id INTO carrera_del_plan
    FROM plan_estudio WHERE id = NEW.plan_estudio_id;
  IF carrera_del_plan <> NEW.carrera_id THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'El plan de estudios no pertenece a la carrera';
  END IF;

  SELECT COUNT(*) INTO faltantes
    FROM materia m
   WHERE m.plan_estudio_id = NEW.plan_estudio_id
     AND m.tipo IN ('REGULAR', 'TCC')
     AND NOT EXISTS (
           SELECT 1
             FROM examen e
             JOIN calificacion cal       ON cal.examen_id = e.id
             JOIN escala_calificacion ec ON ec.codigo = cal.escala_codigo
            WHERE e.materia_id = m.id
              AND cal.alumno_id = NEW.alumno_id
              AND ec.es_aprobatoria = TRUE);
  IF faltantes > 0 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Plan incompleto: hay materias sin aprobar';
  END IF;
END//
DELIMITER ;
```

### 5.4 Inmutabilidad del documento emitido

- **TRIGGER `trg_bloqueo_post_certificado`** (BEFORE UPDATE/DELETE ON
  `calificacion` y `examen`): si ya existe un `certificado` para ese alumno y la
  carrera del examen, rechazar la modificación (el documento legal no cambia).
  Correcciones posteriores = nueva mesa/examen y re-impresión controlada.

### 5.5 Lo que NO necesita trigger

Ningún total ni promedio: al ser 100 % calculados (sección 4), no hay columnas
denormalizadas que mantener sincronizadas.

## 6. Datos semilla (catálogos)

| escala | nota_en_letras | equivalencia | valor | aprobatoria |
|---|---|---|---|---|
| 0 | CERO | Fraude | 0.0 | no |
| 1 | UNO | Aplazado | 1.0 | no |
| 2 | DOS | Aprobado | 2.0 | sí |
| 3 | TRES | Bueno | 3.0 | sí |
| 4 | CUATRO | Distinguido | 4.0 | sí |
| 5 | CINCO | Sobresaliente | 5.0 | sí |
| 5F | CINCO FELICITADO | Sobresaliente Felicitado | 5.0 | sí |

- `periodo_evaluacion`: O = Ordinaria, E = Extraordinaria.
- `modalidad`: P = Presencial, D = Distancia.

## 7. Checklist de implementación

1. Crear tablas y FKs (exportar SQL desde dbdiagram.io).
2. Cargar catálogos semilla (sección 6).
3. Crear CHECKs y triggers (sección 5).
4. Crear vistas `v_certificado_detalle`, `v_certificado_resumen_curso`,
   `v_certificado_totales` con las consultas de la sección 4.
5. Cargar plan de estudios (5 años × 5 bimestres × 2 materias + TCC + pasantía),
   alumno, inscripción, exámenes y calificaciones de prueba.
6. Emitir certificado de prueba y contrastar cada sección contra el documento original.
