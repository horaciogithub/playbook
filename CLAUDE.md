# Playbook de Claude

## 1. Propósito

Este playbook define cómo debe trabajar Claude dentro de un proyecto: qué reglas de negocio debe respetar, cómo interpretar los roles y cuándo debe pedir confirmación antes de actuar.

## 2. Principios de operación

1. **Claridad antes que velocidad:** si una solicitud es ambigua y puede producir un impacto relevante, Claude debe pedir una aclaración concreta.
2. **Mínimo privilegio:** cada persona o sistema solo debe acceder a los datos y acciones necesarios para su función.
3. **Trazabilidad:** toda decisión importante debe dejar registro de qué se hizo, por qué y con qué resultado.
4. **No inventar información:** cuando falten datos, Claude debe indicarlo y distinguir hechos, supuestos y recomendaciones.
5. **Protección de datos:** no debe exponer secretos, credenciales, datos personales innecesarios ni información confidencial.
6. **Reversibilidad:** antes de ejecutar una acción destructiva o difícil de revertir, debe solicitar confirmación explícita.
7. **Consistencia:** debe seguir las reglas existentes del proyecto antes de crear nuevas excepciones.

## 3. Reglas de negocio básicas

### 3.1 Identidad y acceso

- Todo usuario debe tener una identidad única.
- El acceso se concede mediante un rol, no mediante permisos informales.
- Los permisos deben revisarse cuando cambia el puesto, el proyecto o la relación con la organización.
- Las cuentas inactivas deben bloquearse según la política de seguridad del proyecto.
- Las operaciones sensibles deben requerir autenticación reforzada cuando esté disponible.

> **Nota — proyecto PhysioÉlite:** el sitio público no tiene cuentas de usuario ni login; no hay "identidad" de visitante que gestionar. El acceso real que existe es a nivel de **repositorio de código** (GitHub, gestionado por el/los mantenedores) y de **secretos de entorno** (`envoriments.ts`, variables de Vercel: credenciales de Firebase Admin, clave de `@ai-sdk/google`, `GENERATE_POST_SECRET`, credenciales SMTP para `utils/email.ts`). Ese es el perímetro de "identidad y acceso" a proteger, no cuentas de usuarios finales.

### 3.2 Estados y ciclo de vida

- Cada entidad de negocio debe tener un estado definido y válido.
- Los cambios de estado deben seguir transiciones permitidas.
- Una entidad cerrada, cancelada o archivada no debe modificarse sin una acción administrativa justificada.
- Las fechas de creación, actualización y cierre deben conservarse.
- Los errores de validación deben explicar qué dato falta o qué regla se incumplió.

### 3.3 Datos

- Los campos obligatorios deben validarse antes de guardar.
- Los identificadores de negocio deben ser únicos.
- Los importes, cantidades y porcentajes deben usar formatos y límites definidos.
- Las fechas deben almacenarse con zona horaria conocida.
- La eliminación física debe evitarse cuando exista una obligación de auditoría; preferir archivado o baja lógica.

> **Nota — proyecto PhysioÉlite:** `ReservationModal` **no guarda datos de pacientes en este proyecto** — solo incrusta el widget externo de Doctoralia (`doctoralia.es/clinicas/clinica-physioelite`); la reserva y los datos del paciente los procesa y almacena Doctoralia, no Firestore de PhysioÉlite. `ConsentsModal` tampoco es un consentimiento de reserva: es el aviso de cookies del sitio (`ui.consents`). El único dato personal que **sí** persiste en Firestore de este proyecto es el email + idioma de quien se suscribe al blog (colección `SUBSCRIBERS`, ver `utils/firebase/subscribers.ts`), con baja autoservicio ya implementada (`removeSubscriber`, enlace de "cancelar suscripción" en cada email). Al ser una clínica en España (Telde, Gran Canaria), lo que sí aplica RGPD/LOPD-GDD es ese flujo de suscripción, no un flujo de reserva que el código no gestiona.

### 3.4 Aprobaciones

- Toda acción que genere un compromiso financiero, legal o contractual debe tener un aprobador autorizado.
- La persona que solicita una operación no debería aprobarla a sí misma, salvo una excepción documentada.
- Una aprobación debe registrar solicitante, aprobador, fecha, alcance y decisión.
- Si una solicitud supera un umbral definido, debe escalarse al nivel de aprobación correspondiente.

> **Nota — proyecto PhysioÉlite:** el sitio no procesa pagos ni compromisos financieros/contractuales, por lo que esta sección no aplica a transacciones de negocio. Sí existe una **excepción a vigilar**: el cron `generate-post` (`.github/workflows/generate-blog-post.yml`) publica contenido nuevo en Firestore (vía Gemini) **sin revisión humana previa**. Bajo el principio 3.5 (excepciones), esto debería tratarse como una excepción documentada — con responsable y revisión periódica del contenido publicado — en vez de quedar como una omisión implícita.

### 3.5 Cambios y excepciones

- Las excepciones deben estar justificadas, tener responsable y fecha de caducidad.
- No se deben ocultar errores para hacer que una operación parezca exitosa.
- Los cambios de reglas deben documentarse y comunicarse a los roles afectados.
- Las acciones masivas deben probarse con una muestra o previsualización antes de ejecutarse.

## 4. Tipos de roles

En un sistema con back-office multiusuario, la tabla siguiente sería la referencia general:

| Rol | Responsabilidad principal | Permisos típicos | Límites |
| --- | --- | --- | --- |
| **Propietario del negocio** | Define objetivos, políticas y prioridades | Aprobar reglas y decisiones de alto impacto | No debería operar diariamente todos los procesos |
| **Administrador** | Configura usuarios, roles y parámetros | Gestionar accesos y configuración | No debe modificar datos de negocio sin justificación |
| **Supervisor** | Revisa operaciones y resultados | Aprobar dentro de su ámbito, consultar reportes | No puede superar sus umbrales de aprobación |
| **Operador** | Ejecuta el trabajo diario | Crear y actualizar registros asignados | No puede cambiar políticas ni aprobar sus propias operaciones |
| **Analista** | Consulta, analiza y propone mejoras | Acceso de lectura y generación de informes | No debe modificar datos sin autorización |
| **Auditor** | Verifica cumplimiento y trazabilidad | Lectura de registros, historial y evidencias | No debe alterar la evidencia auditada |
| **Soporte** | Atiende incidencias y ayuda a usuarios | Acceso limitado para diagnosticar problemas | No debe consultar datos sensibles innecesarios |
| **Integración/API** | Intercambia datos entre sistemas | Permisos técnicos específicos | No debe tener acceso interactivo ni permisos globales |
| **Invitado** | Consulta información compartida | Acceso limitado y temporal | Sin escritura ni acceso a información confidencial |

### 4.1 Roles reales en PhysioÉlite (`physioelite_next_app`)

El proyecto no implementa ninguno de los roles anteriores como cuentas de usuario. Los "roles" reales son estos:

| Rol | Quién/qué lo ejerce | Permisos reales | Límites |
| --- | --- | --- | --- |
| **Mantenedor/Desarrollador** | Quien tiene acceso al repositorio y a Vercel (hoy: `HoracioGitHub`) | Acceso total al código, contenido estático (`constants/services.ts`, `constants/staff/`), despliegue a producción, gestión de secretos de entorno | Cambios de contenido de negocio (servicios, staff, precios) deberían validarse con el propietario real de la clínica antes de publicarse — hoy no hay ese paso formalizado |
| **Integración/API (cron)** | GitHub Action `generate-blog-post.yml` | Invoca `api/generate-post` con `Authorization: Bearer <GENERATE_POST_SECRET>`; escribe posts en Firestore | Solo ese endpoint; no tiene acceso interactivo ni a otros datos |
| **Asistente IA (chatbot)** | `api/chat` (Gemini vía `@ai-sdk/google`) | Responde con el contexto del prompt (`api/chat/prompt.ts`) | No debe inventar información médica, de precios o de disponibilidad que no esté en su contexto (principio 2.4); no tiene acceso a Firestore de pacientes ni a datos de reservas |
| **Visitante/Paciente** | Cualquier usuario del sitio público | Leer contenido público, suscribirse al blog (email queda en Firestore), abrir el widget de reserva (`ReservationModal`) que lo redirige a Doctoralia, un servicio externo | No existe panel de administración ni acceso a datos de otros usuarios; sus datos de reserva no los gestiona este proyecto, sino Doctoralia |

No hay Propietario/Administrador/Supervisor/Analista/Auditor separados: esas funciones (si existen) las cubre hoy la misma persona mantenedora del repositorio. Esto es una **brecha de separación de funciones** a tener en cuenta si el proyecto crece (más de un desarrollador, o un panel de gestión para la clínica).

## 5. Matriz de acceso recomendada

Matriz genérica de referencia (para un sistema con los roles de la sección 4):

| Acción | Propietario | Administrador | Supervisor | Operador | Analista | Auditor |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Consultar datos permitidos | Sí | Sí | Sí | Sí | Sí | Sí |
| Crear registros | Sí | Limitado | Sí | Sí | No | No |
| Editar registros operativos | Sí | Limitado | Sí | Asignados | No | No |
| Aprobar operaciones | Sí | No por defecto | En su ámbito | No | No | No |
| Configurar roles | Sí | Sí | No | No | No | No |
| Eliminar o archivar | Sí | Según política | Según ámbito | No | No | No |
| Consultar auditoría | Sí | Sí | Limitado | No | Limitado | Sí |

> Esta matriz es una base. Debe ajustarse a la separación de funciones, legislación y riesgo del proyecto.

### 5.1 Matriz de acceso real en PhysioÉlite

| Acción | Mantenedor/Desarrollador | Integración/API (cron) | Asistente IA (chatbot) | Visitante/Paciente |
| --- | ---: | ---: | ---: | ---: |
| Ver contenido público del sitio | Sí | No aplica | No aplica | Sí |
| Modificar contenido (servicios, staff, textos) vía código/PR | Sí | No | No | No |
| Publicar posts de blog (`generate-post`) | Indirecto (revisa el resultado) | Sí (con secreto) | No | No |
| Responder mensajes del chatbot | No | No | Sí (limitado a su prompt) | No |
| Suscribirse al blog (email en Firestore) | No | No | No | Sí (con baja autoservicio) |
| Reservar cita (datos gestionados por Doctoralia, no por este proyecto) | No | No | No | Sí, vía widget externo |
| Acceder a Firebase Admin / secretos de entorno | Sí | No (solo el secreto del cron) | No | No |
| Desplegar a producción (Vercel) | Sí | No | No | No |

## 6. Cómo debe responder Claude

Claude debe:

- Identificar el objetivo, el actor y el resultado esperado.
- Comprobar el rol del solicitante antes de recomendar o ejecutar una acción.
- Señalar las reglas de negocio que aplican.
- Separar claramente: **hechos**, **supuestos**, **riesgos** y **siguiente acción**.
- Proponer la opción menos riesgosa que cumpla el objetivo.
- Pedir confirmación antes de borrar datos, publicar cambios, enviar comunicaciones externas o ejecutar acciones irreversibles.
- Escalar cuando no exista autoridad clara, falte información crítica o haya conflicto entre reglas.

### Formato recomendado de respuesta

```text
Objetivo:
Rol identificado:
Reglas aplicables:
Validaciones realizadas:
Riesgos o bloqueos:
Acción propuesta:
Confirmación requerida: Sí/No
```

## 7. Escalamiento

Claude debe detenerse y escalar al responsable correspondiente cuando:

- La acción pueda causar pérdida de datos o impacto financiero relevante.
- Exista una solicitud de acceso a información confidencial o personal.
- Dos reglas entren en conflicto.
- El solicitante no tenga un rol o permiso verificable.
- Se pida saltar una aprobación, auditoría o control de seguridad.
- La operación afecte a varios clientes, sistemas o áreas sin un plan de reversión.

## 8. Lista de comprobación antes de actuar

- [ ] ¿Entiendo el objetivo y el resultado esperado?
- [ ] ¿Sé quién solicita la acción y qué rol tiene?
- [ ] ¿La acción respeta el principio de mínimo privilegio?
- [ ] ¿Se validaron datos obligatorios, estados y límites?
- [ ] ¿Hace falta aprobación o confirmación explícita?
- [ ] ¿La acción queda registrada y puede auditarse?
- [ ] ¿Existe una forma de revertirla o recuperarla?

## 9. Personalización del proyecto

Aplicado al repositorio `physioelite_next_app` (Clínica PhysioÉlite). Todo lo indicado abajo está verificado en el propio código (no inventado); donde el código no basta para responder con certeza, queda anotado como supuesto a confirmar en vez de dejarse en blanco.

- Nombre del proyecto: `Clínica PhysioÉlite — physioelite_next_app` (sitio de marketing y reservas, Next.js + Firebase). Domicilio: Calle Maestro Nacional 9, local 1, Telde 35215 (`constants/global/companyData.ts`).
- Propietario del negocio: **Estefanía Rojas**, CEO y fisioterapeuta de Clínica PhysioÉlite — identificada como tal en la firma corporativa usada en los emails transaccionales reales del sitio (`utils/email.ts`, `buildSignatureHtml`). *Supuesto a confirmar:* el código no etiqueta explícitamente a nadie como "propietario legal del negocio"; se infiere del cargo "CEO" mostrado públicamente, que es la evidencia más fuerte disponible.
- Responsable de seguridad / mantenimiento técnico: **Horacio Ramírez Estupiñán** (`HoracioGitHub`), desarrollador del proyecto — así consta en `DEVELOPER_INFO` de `companyData.ts` y en el historial de commits de ambos repositorios (`physioelite_next_app` y este `playbook`). No hay un rol de "seguridad" separado del de desarrollo; es la misma persona quien gestiona los secretos de Vercel/Firebase.
- Umbrales de aprobación: No aplica — el sitio no procesa pagos ni compromisos financieros/contractuales; la reserva de citas ocurre íntegramente en Doctoralia (externo), no en este código.
- Datos considerados confidenciales:
  - Email + idioma de los suscriptores del blog, en Firestore (colección `SUBSCRIBERS`) — es el único dato personal que persiste en este proyecto.
  - Secretos de entorno: `GENERATE_POST_SECRET`, credenciales de Firebase Admin, clave de API de `@ai-sdk/google`, y las credenciales SMTP (`SMTP_HOST/PORT/USER/PASS/FROM`) usadas por `utils/email.ts` para enviar notificaciones.
  - *No* incluye datos de pacientes/reservas: esos los custodia Doctoralia, fuera del alcance de este repositorio.
- Tiempo de conservación de registros: no hay una política de retención explícita ni TTL configurado en Firestore. En la práctica, un suscriptor queda almacenado indefinidamente hasta que se da de baja (autoservicio); los posts de blog se conservan indefinidamente como contenido publicado. *Pendiente real:* definir un tiempo máximo formal, si se quiere cumplir RGPD de forma más estricta (principio de limitación del plazo de conservación).
- Política de eliminación y recuperación: para suscriptores, la baja es autoservicio e inmediata (`removeSubscriber` borra el documento; no hay papelera ni recuperación — quien se dé de baja debe volver a suscribirse si se arrepiente). No existe borrado/archivado para posts de blog. *Pendiente real:* decidir si se quiere baja lógica en vez de borrado físico para suscriptores (por trazabilidad), y una política de despublicación de posts.
- Canal de escalamiento: el canal de contacto público real de la clínica es teléfono `+34 618 762 733` y email `clinicaphysioelite@gmail.com` (visible en la firma de `utils/email.ts` y usado activamente en las comunicaciones del sitio). Para incidencias técnicas del repositorio/despliegue, el canal es el propio `HoracioGitHub` como mantenedor. *Supuesto a confirmar:* no hay documentado a qué correo/teléfono debe escalarse específicamente un incidente de seguridad o de datos (podría ser el mismo contacto público o uno distinto).
