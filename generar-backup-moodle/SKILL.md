---
name: generar-backup-moodle
description: Genera una copia de seguridad de curso Moodle (.mbz) compatible con Aules (Moodle 5.2) a partir de una asignatura de informatica-eso-bat.
---

# Generar copia de seguridad de Moodle (.mbz) para Aules (Moodle 5.2)

Cuando se ejecute este skill, tu objetivo será generar un archivo de copia de seguridad de curso de Moodle (`.mbz`) compatible e importable directamente en la plataforma **Aules** (basada actualmente en **Moodle 5.2.1+**), a partir del contenido de una asignatura de la web [informatica-eso-bat].

---

## 1. Especificación de la Copia de Seguridad

En esta sección se definen las reglas y el comportamiento estándar con el que se construye el aula virtual de Moodle:

### 1.1. Origen y Enlaces Web
* **Web base**: Las asignaturas se alojan y consultan en `https://dlopezcastellote.dev/informatica-eso-bat`.
* El código fuente de las asignaturas se encuentra en la carpeta [asignaturas/] (por ejemplo: `asignaturas/digitalizacion`, `asignaturas/psiri`, etc.).

### 1.2. Estructura de Secciones (Temas)
* **Sección 0**: Cabecera general de Moodle (reservada para avisos/general).
* **Sección 1 a N**: Cada tema de la asignatura ocupa exactamente una sección de Moodle.
* **Botón web en cada sección**: Al inicio de cada sección/tema se añade un recurso tipo **Etiqueta (label)** con un botón estilizado centrado que abre el tema en la web:
  ```html
  <p style="text-align: center;">
    <a style="display: inline-block; padding: 14px 28px; font-family: 'Segoe UI', Roboto, sans-serif; font-size: 16px; font-weight: 600; letter-spacing: 0.5px; color: #ffffff; background-color: #0076ff; text-decoration: none; border-radius: 8px; box-shadow: 0 4px 6px rgba(0, 118, 255, 0.2); transition: background-color 0.2s ease;"
       href="https://dlopezcastellote.dev/informatica-eso-bat/asignaturas/{asignatura}/{tema}/" 
       target="_blank" 
       rel="noopener noreferrer">
       TEMA {N} (WEB)
    </a>
  </p>
  ```

### 1.3. Estructura de Actividades (Tareas de Entrega)
* Cada actividad detectada en la web se traduce a un recurso de Moodle de tipo **Tarea (`assign`)** donde el alumnado puede realizar entregas de archivos.
* **Descripción de la tarea (`intro`)**: Consiste únicamente en un enlace directo a la actividad en la web:
  ```html
  <p><strong>➡️ACTIVIDAD</strong>: <a href="https://dlopezcastellote.dev/informatica-eso-bat/asignaturas/{asignatura}/{tema}/{actividad}/" target="_blank" rel="noopener">https://dlopezcastellote.dev/informatica-eso-bat/asignaturas/{asignatura}/{tema}/{actividad}/</a></p>
  ```
* **Configuración por defecto de la tarea**:
  - Calificación máxima: `10`, calificación para aprobar: `5`.
  - Envío de archivos activado (`assignsubmission_file` enabled=1).
  - Número máximo de archivos subidos: `20`.
  - Tamaño máximo de archivo: `50MB` (52428800 bytes).
* **Importación automática de Rúbricas (`rubrica.csv`)**:
  - Si en la carpeta de la actividad existe un archivo `rubrica.csv` (generado habitualmente con el skill `/convertir-rubrica-csv`), el generador lo detecta y lo importa automáticamente.
  - La tarea se configura en Moodle con **calificación avanzada por rúbrica** activa (`activemethod: rubric`, estado `Ready` / 20).
  - Los criterios y niveles se trasladan con sus descripciones y puntuaciones exactas.
  - Si la tarea está duplicada por grupos (ej. `DIG1` y `DIG2`), la rúbrica se asocia de forma independiente a cada una de las tareas creadas para que ambos grupos dispongan de la rúbrica de calificación.
* **Mostrar descripción en la página del curso (`show_activity_description`)**:
  - En Moodle, cada tarea dispone de la opción *"Muestra la descripción en la página del curso"* (`<showdescription>` en `module.xml`).
  - Se puede configurar dentro de cada tema en la lista `topics` mediante `"show_activity_description": true` o `false` (por defecto es `false`, lo habitual para no sobrecargar visualmente el curso).
  - También es posible definirlo a nivel raíz del JSON como valor global por defecto para todos los temas.
* **Visibilidad de Secciones y Actividades (`visible` / `activities_visible`)**:
  - **Ocultar un tema/sección completo**: En `topics`, `"visible": false` (o `0`) genera la sección oculta para el alumnado (`<visible>0</visible>` en `section.xml`).
  - **Ocultar actividades por defecto en un tema**: En `topics`, `"activities_visible": false` (o `0`) hace que todas las tareas del tema se generen ocultas (`<visible>0</visible>` en `module.xml`). También puede definirse a nivel global en la raíz del JSON.
  - **Ocultar una actividad específica**: Dentro de `activities` (tanto en la sección general como en un tema), `"visible": false` (o `0`) oculta esa actividad individualmente.
* **Archivos del resumen del curso (`overviewfiles`)**:
  - En la configuración del curso de Moodle, el campo *"Archivos del resumen del curso"* muestra la imagen del curso en el Dashboard / Área personal de los alumnos y profesores.
  - El script localiza automáticamente la imagen del banner configurada en la sección general (o por convención en `archivos/imagenes/*.png`, o explícitamente en el JSON mediante `"course_image": "ruta/o/nombre.png"` dentro de `"course"` o en la raíz).
  - La imagen se empaqueta con su hash SHA-1 en el pool de archivos de Moodle (`files/`) y se registra en `files.xml` y `course/inforef.xml` en el área `overviewfiles` del componente `course`.

### 1.4. Cuestionarios y Banco de Preguntas (`.gift`)
* Permite incorporar a cualquier tema un cuestionario de autoevaluación o examen a partir de un archivo con formato **GIFT** (extensión `.gift`, por ejemplo `cuestionario_tema1_sistemas_operativos.gift`).
* **Banco de preguntas (`questions.xml`)**:
  - Todas las preguntas contenidas en el archivo `.gift` se incorporan automáticamente al Banco de Preguntas del curso de Moodle.
  - **Categoría configurable**: Cada cuestionario puede especificar su categoría en el banco (`category`). Si no se define, se asigna automáticamente al título del tema.
  - **Versionado Moodle 4.x/5.x**: Se crea la jerarquía oficial con categoría superior `top`, subcategorías, `question_bank_entry`, `question_version` y referencias en los slots del cuestionario con versión dinámica (`$@NULL@$`), garantizando que siempre se utiliza y actualiza la versión más reciente del banco.
  - **Tipos de preguntas soportados**: Opción múltiple (`multichoice` con respuesta simple o respuestas ponderadas porcentualmente), Verdadero/Falso (`truefalse`), Emparejamiento (`match`), Respuesta corta (`shortanswer`) y Ensayo/Desarrollo (`essay`).
* **Opciones por defecto del cuestionario (configurables en el JSON)**:
  - **Sin retroalimentación hasta el cierre**: Por defecto, las máscaras de revisión (`reviewattempt`, `reviewcorrectness`, `reviewmarks`, `reviewspecificfeedback`, `reviewgeneralfeedback`, `reviewrightanswer`, etc.) se fijan en `16` (`AFTER_CLOSE`), de modo que no se muestra ningún tipo de puntuación, solución o retroalimentación mientras el cuestionario esté abierto o durante los intentos.
  - **Tiempo límite**: Por defecto `20` minutos (`time_limit: 20` -> 1200 segundos).
  - **Intentos permitidos máximos**: Por defecto `1` intento (`max_attempts: 1` -> `<attempts_number>1</attempts_number>`).
  - **Preguntas en orden aleatorio**: Por defecto barajadas (`shuffle_questions: true` -> `<shufflequestions>1</shufflequestions>`).
  - **Respuestas barajadas**: Por defecto activado (`shuffle_answers: true`).
  - **Calificación máxima**: Por defecto `10.0` puntos (escalado automático sobre la suma total de puntos de las preguntas).

### 1.5. Gestión y Duplicación por Grupos (Clases)
* Si una asignatura tiene varios grupos (por ejemplo en Digitalización: `DIG1` y `DIG2`):
  - Cada grupo representa una clase distinta que puede tener ritmos o fechas límite de entrega diferentes.
  - Las tareas y cuestionarios se **duplican automáticamente** (una actividad para `DIG1` y otra para `DIG2`).
  - Cada actividad duplicada incluye su condición de disponibilidad (`availability`) restringida exclusivamente a ese grupo:
    `{"op":"&","c":[{"type":"group","id":<group_id>}],"showc":[true]}`.
  - Los grupos quedan definidos en `groups.xml` y referenciados en `course/inforef.xml` para que Moodle cree los grupos al restaurar.
* Si no se configuran grupos, las actividades se crean una única vez sin restricciones de disponibilidad.

### 1.6. Archivo de Configuración de Asignatura
Dentro de la carpeta de la asignatura puede existir un archivo llamado `config_backup_moodle_<asignatura>.json` (o `config_backup_moodle.json`) para personalizar la copia. Ejemplo:
```json
{
  "course": {
    "fullname": "4ESO - Digitalización",
    "shortname": "4ESO-DIG",
    "idnumber": "4ESO_DIG_2025_26",
    "summary": "Descripción opcional del curso"
  },
  "web_base_url": "https://dlopezcastellote.dev/informatica-eso-bat/asignaturas/digitalizacion",
  "groups": [
    { "id": 88658, "name": "DIG1" },
    { "id": 88659, "name": "DIG2" }
  ],
  "duplicate_activities_per_group": true,
  "topic_button": {
    "style": "display: inline-block; padding: 14px 28px; font-family: 'Segoe UI', Roboto, sans-serif; font-size: 16px; font-weight: 600; letter-spacing: 0.5px; color: #ffffff; background-color: #0076ff; text-decoration: none; border-radius: 8px; box-shadow: 0 4px 6px rgba(0, 118, 255, 0.2); transition: background-color 0.2s ease;"
  },
  "general_section": {
    "activities": [
      {
        "type": "label",
        "name": "Banner Asignatura",
        "content": "<h3 style=\"text-align: center; color: blue\"><strong><img src=\"https://dlopezcastellote.dev/informatica-eso-bat/archivos/4ESO-DIGITALIZACION.png\" alt=\"4ESO DIGITALIZACIÓN\" width=\"999\" height=\"333\" role=\"presentation\" class=\"img-fluid atto_image_button_text-bottom\"></strong></h3>"
      },
      {
        "type": "label",
        "name": "Profesor y Contacto",
        "content": "<table style=\"border-collapse: collapse; margin: 0 auto\">...</table>"
      },
      {
        "type": "forum",
        "name": "Avisos y noticias",
        "intro": "Anuncis i notícies generals",
        "forum_type": "news"
      },
      {
        "type": "url",
        "name": "Criterios de calificación",
        "external_url": "https://dlopezcastellote.dev/informatica-eso-bat/archivos/Criterios-de-calificaci%C3%B3n-ESO-y-Bachillerato.pdf"
      }
    ]
  },
  "topics": [
    {
      "folder": "sistemas-operativos",
      "title": "Tema 1: Sistemas Operativos",
      "button_text": "TEMA 1 (WEB)",
      "show_activity_description": false,
      "visible": true,
      "activities_visible": true,
      "quizzes": [
        {
          "name": "Cuestionario Tema 1: Sistemas Operativos",
          "gift_file": "cuestionario_tema1_sistemas_operativos.gift",
          "category": "Tema 1: Sistemas Operativos",
          "time_limit": 20,
          "max_attempts": 1,
          "shuffle_questions": true,
          "shuffle_answers": true,
          "feedback_mode": "after_close"
        }
      ]
    },
    {
      "folder": "documentos-digitales",
      "title": "Tema 2: Documentos digitales",
      "button_text": "TEMA 2 (WEB)",
      "visible": false
    }
  ]
}
```
*Si `topics` no se define en el JSON, el generador autodescubre los temas leyendo el archivo `index.md` de la asignatura y las actividades escaneando las subcarpetas del tema.*

---

## 2. Instrucciones de Ejecución del Skill

Cuando el usuario pida generar un backup (ej: *"genera el backup de digitalizacion"* o proporcione una ruta de asignatura):

### Paso 1: Identificar la Asignatura
1. Determinar la ruta de la carpeta de la asignatura dentro de `informatica-eso-bat/asignaturas/` (ej. `digitalizacion`, `psiri`, `piari`, etc.).
2. Si el usuario no especificó la asignatura o la ruta no existe, preguntar amablemente qué asignatura desea empaquetar.

### Paso 2: Revisar o Crear el Archivo de Configuración
1. Comprobar si existe `config_backup_moodle_<asignatura>.json` dentro de la carpeta de la asignatura.
2. Si **no existe**, ofrecer crear uno con los datos deducidos de `index.md` o proceder con los valores por defecto (preguntando si existen grupos de clase específicos como DIG1, DIG2, etc.).

### Paso 3: Ejecutar el Generador de Backup
Ejecutar el script generador mediante la herramienta de comandos:
```bash
python scripts/generate_moodle_backup.py <ruta_asignatura>
```
*(El archivo se generará automáticamente con fecha y hora: `informatica-eso-bat/archivos/backups/backup_moodle_<asignatura>_<YYYYMMDD_HHMMSS>.mbz`. Opcionalmente se puede especificar `--output <ruta_salida.mbz>`).*

### Paso 4: Validar y Notificar al Usuario
1. Verificar que el comando terminó con código de salida `0` y que el archivo `.mbz` se ha generado correctamente.
2. Comprobar el tamaño y el resumen de temas y actividades generadas.
3. Informar al usuario de:
   - Ruta completa del archivo `.mbz` generado listo para importar en Aules.
   - Cantidad de temas/secciones creadas y enlaces web generados.
   - Lista de actividades y si fueron duplicadas por grupos.
   - Instrucciones breves de restauración en Aules (*Restaurar curso -> Subir archivo .mbz -> Restaurar como curso nuevo o fusionar*).
