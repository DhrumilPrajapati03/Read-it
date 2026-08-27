Part 1 — The Training Business Layer: full doctype inventory
The layer spans four apps. This is the whole map, grouped by role in the lifecycle:

A. Funding & planning (the "what we're allowed to teach")
DocType	App	Role
Planes Formativos	education	The funded programme. States: Planificado → En Curso → Ejecutado / Anulado / Incidentado / Previsto. Holds subvención_concedida, subvención_ejecutada, alumnado_concedido, plazo_de_ejecución
Planes	education	Plan category catalogue
Configuracion Plan / Version Configuracion Plan	education	Versioned plan rules
Requisitos Plan Formativo / Tipo de Requisito	education	Documentary requirements per plan
Oferta Convocada	custom_education	The public tender offer — codigo_plan, codigo_accion, especialidad, horas, subvencion_por_accion, alumnos, provincia, modalidad
Acciones Formativas Concedidas	custom_education	Granted training actions
Convocatoria (via Academic Year)	education	Funding call/round
Especialidades Formativas, Familias Formativas, Area Profesional	education	SEPE official catalogue
Certificados de Profesionalidad, cp, cp_denominacion, cpv	both	Certificate-of-professionalism catalogue
Modulo Profesional, Bloque Formativo, Modalidad Formativa	education	Curriculum structure
Ambito Sectorial, Sector, Sectores Expediente, Convenio	custom_education	Sectoral scope
Colectivos Prioritarios	custom_education	Priority target groups (drives funding eligibility)
Porcentajes Bonificada, Porcentaje Bonificada Cliente	custom_education	FUNDAE subsidy rates
B. Execution structure (the "what we're actually running")
DocType	Role
Program	Expediente — the case file grouping courses under a plan
Course	Acción Formativa (AF) — 89 fields. expediente, state, center, aula, start/end_date, horas_efectivas, horas_descontadas, tipo_curso (Teoría/Especialidad/Estancia Empresa), Moodle virtual_class, milestone dates (15_percent, 23_percent, 75_percent, 5º_day)
Program Course / Program Course Link / Cursos Hija	Course composition
Expediente Linea Accion Formativa	Expediente ↔ AF line
Imparticion de Certificado	Certificate delivery — expediente, codigo_certificado, grado_imparticion (A/B/C/D), cursos_generados, unified calendar
Cursos Generados	Courses auto-generated from a certificate
Course Schedule, Course Time Slot, Course Theory Schedule, Anexos Time Slot	Timetabling
Course Scheduling Tool	Bulk schedule generator
Aula, Cursos Aulas, Dotacion Aulas, Imagenes Aula	Classroom — physical/virtual, equipment, photos
Room	Training centre (center on Course)
Requisitos Centros / Requisitos Centros Estados	Centre accreditation
Instructor, Instructor Curso, Instructor Habilitacion, Instructor Log	Teaching staff + accreditation to teach a speciality
Student Group, Grupo Estudiantes Curso, Student Group Creation Tool, Student Batch Name	Cohorts
C. Students
DocType	Role
Student Applicant	The application. application_status: Applied → Approved → Rejected → Admitted. Carries the full FUNDAE dataset: employment_situation, grupo_cotizacion, tipo_de_contrato, nivel_estudios, ocupacion, categoría, desempleado_de_larga_duración, pyme, tamaño_empresa, parado_seleccion_sepe, doc_type (DNI/NIE/Pasaporte), sector, convenio
Student	The enrolled person (student_applicant link back)
Student Admission / Student Admission Program	Admission campaigns
Program Enrollment (+ Course, Fee children)	Matriculación — student ↔ expediente
Course Enrollment	student ↔ single AF. Fields: program_enrollment, student, course, enrollment_date
Program Enrollment Tool (+ Student child)	Bulk enrolment
Estudiantes y Acciones / Estudiantes y Acciones Acciones	Student↔action matrix
Estudiantes Curso, Estudiantes Grupo Estudiantes	Roster children
Student Attendance + Student Attendance Tool	Asistencia — the single most audited artefact
Student Log, Comunicacion Estudiante, Comunicaciones Estudiante Comunicaciones	Contact history
Incidencia Estudiante / Incidencias Estudiante Incidencias	Issues
Student Leave Application	Absence justification
Encuesta de Calidad	Quality survey — mandatory for justification
Student Portal Auth / Student Portal OTP	Student portal login (bcrypt + OTP, lockout, rate limit)
Interes Profesional Alumno, Perfiles Candidatos	Profiling
Trae un Amigo / Trae un Amigo Detail	Referral scheme
Guardian, Student Guardian, Student Sibling, Student Category, Student Language	Standard education extras
D. Assessment & certification
DocType	Role
Assessment Plan / Assessment Plan Criteria	Exam design
Assessment Criteria / Group / Course Assessment Criteria	Rubrics
Assessment Result / Assessment Result Detail / Assessment Result Tool	Grades
Grading Scale / Grading Scale Interval	Scoring
Quiz, Question, Options, Quiz Question, Quiz Result, Quiz Activity	Online testing
Certificados Centros de Formación	Centre-issued certificates
Certificaciones Empresa Centro	Company/centre certifications
Student Report Generation Tool	Report cards
— generar_diplomas.py (2,398 lines)	Diploma/certificate generation: Word/PDF templates by category (estatal/FORTRA/ocupados), digital signature application, email dispatch with tracked Email Queue status, Drive folder storage
E. Classroom engagement (aula_* — the tutor/dynamisation layer)
Aula Gamificacion Medalla, Aula Gamificacion Movimiento, Aula Instructor Gamificacion, Avatar Library, Aula Tarea Dinamizacion, Aula Tarea Colectiva (+ Compartida), Aula Tutor Comunicacion, Aula Seguimiento Hito, Aula Pedagogia Ticket, Aula Notificacion

Instructors earn points/levels/badges/streaks from closing ToDos (todo_on_update_gamification hook). Avatar tier scales with level.

F. Demand generation
Solicitud Formacion Empresa + Solicitud Formacion Linea (B2B training requests), Censo Empresa Centro, Customer Employees, OV Employees, Pack Comercial (+ Accion), Job Offer Course, JobOffer Time Slot, Marketing Campaign, Content Studio *, Marketing RRSS Publication

G. Compliance automation
RPA Process Definition, RPA Settings, RPA Trigger Rule, RPA Field Mapping, RPA Document Mapping, RPA Execution Log, RPA Execution Evidence, RPA Execution Document Result — bots that file into SEPE/FUNDAE/regional portals and keep evidence.

Part 2 — The end-to-end flow

① FUNDING CALL
   Academic Year (Convocatoria) published by SEPE/FUNDAE/CCAA
   └─> Oferta Convocada imported  (utils/oferta_convocada_excel.py)
       codigo_plan · especialidad · horas · alumnos · subvención_por_accion

② BID / GRANT
   Planes Formativos created  (estado = Previsto)
   ├─ Nombre_Planes.validate → auto-naming from category+year+increment
   ├─ apply_auto_lock_on_insert → plan_bloqueo locks edits once live
   └─ Acciones Formativas Concedidas records what was actually granted
       estado → Planificado

③ EXPEDIENTE + AF STRUCTURE
   Program (expediente) created under the plan
   └─> Course (Acción Formativa) per granted action
       hooks fire: convocatoria_sync · imparticion_sync · enforce_plan_lock
                   course_image_generator (AI cover) · marketing sync
   Imparticion de Certificado → auto-generates Cursos Generados for CP courses
   Room (centre) + Aula assigned; Instructor Habilitacion checked

④ DEMAND / RECRUITMENT  ── three entry routes, see Part 3
   Marketing Campaign · Content Studio · RRSS publication · Trae un Amigo

⑤ APPLICATION
   Student Applicant  (application_status = Applied)
   ├─ validate → address_cp.validate_student_applicant (postcode↔province)
   ├─ after_insert → send_email_on_plan_telefono
   ├─ colectivos_prioritarios_deteccion → auto-flags priority group
   └─ fundae_seleccion / parado_seleccion_sepe for unemployed quota

⑥ SELECTION
   status → Approved → Admitted
   cupo_af.py enforces the funded seat cap (alumnos on Oferta Convocada)
   Student created, student_applicant linked back

⑦ ENROLMENT
   Program Enrollment (expediente-level)
   └─> Course Enrollment per AF   [Program Enrollment Tool for bulk]
   Student Group / Grupo Estudiantes Curso built
   Moodle: moodle_scorm_connector + virtual_class provisioning

⑧ DELIVERY
   Course.state → En Curso
   Course Schedule / Course Time Slot drive the calendar
   Student Attendance per session  (api/student_attendance.py)
     · guardar_asistencia · subir_justificante (absence proof)
     · get_alertas_asistencia → drop-out early warning
   Milestone dates tracked: 15% · 23% · 75% · 5º day
   aula_* dinamización: tutor comms, collective tasks, hitos, gamification
   Entrega Material AF · Dotacion Aulas evidence captured

⑨ ASSESSMENT
   Assessment Plan → Assessment Result (per criteria, per grading scale)
   Quiz / LMS Quiz Submission for online
   Encuesta de Calidad — mandatory satisfaction survey (SEPE print format exists)

⑩ CLOSURE
   Course.state → Ejecutado · fecha_fin_real · horas_efectivas/descontadas
   Estudiantes y Acciones finalised

⑪ CERTIFICATION
   generar_diplomas.py:
     resolve category (estatal / FORTRA / ocupados) from plan+course text
     → pick Word/PDF template per categoria + estado_matricula
     → render per student
     → apply digital signatures (signature_integration / nebula)
     → email with embedded logo, track Email Queue status
     → store in Drive folder per course
   Certificados Centros de Formación / LMS Certificate issued

⑫ JUSTIFICATION  ── hands off to justificacionatu
   Imputaciones de Nomina: instructor hours → this AF (Concepto Directo/Indirecto)
   Movimiento de Imputacion: invoices → centre + course
   Certificados de imputación de personal generated
   RPA bots file the dossier into the government portal, storing
   RPA Execution Evidence as proof of submission
   Planes Formativos.estado → Ejecutado · subvención_ejecutada filled
Scheduled jobs that police this (custom_education/hooks.py):

daily_requisitos_check — missing documentary requirements
daily_actualizar_estados_planes — auto-advance plan states
justification_execution_reminder, weekly_execution_deadline_notice — deadline alarms
notificar_responsables_planes (weekly)
Part 3 — How a student actually applies
There are three distinct routes, not one:

Route 1 — B2B (company-driven). A company logs into the education portal, browses Oferta Convocada filtered by plan/province/speciality, and files a Solicitud Formacion Empresa with Solicitud Formacion Linea rows (seats requested per action, compromiso_parcial/compromiso_completa). ATU reviews it (_ensure_can_review_solicitud, solicitud_formacion_valoracion.py), checks coverage (solicitud_formacion_cobertura.py), and their employees (Customer Employees / OV Employees) become Student Applicant records.

Route 2 — Individual self-application. Via the public education SPA at /education/* (served by professional_certificate_creation_tool). Creates a Student Applicant directly. Auth for returning students is not a Frappe User — it's a custom stack: Student Portal Auth (bcrypt hash, failure counter, lockout) + Student Portal OTP (emailed code, hourly send cap, purpose-scoped) in portal_auth.py.

Route 3 — SEPE/administrative referral + referral scheme. Unemployed candidates arrive via SEPE selection (parado_seleccion_sepe, fundae_seleccion.py), or through Trae un Amigo (existing student refers a friend, tracked with detail rows for reward attribution).

All three converge on Student Applicant → selection → Student → Program Enrollment → Course Enrollment.

Part 4 — Your voice assistant today: exact capability audit
Architecture (voice_command.py, 2,203 lines):


process_voice_command(command, context)
  ├─ 1. _try_llm()      → Groq API, tools=_get_tools(), tool_choice=auto
  ├─ 2. _try_analytics() → regex fallback
  ├─ 3. _try_navigation()→ regex fallback over DOCTYPE_MAP/REPORT_MAP/PAGE_MAP
  └─ 4. canned help text (ES/EN via _is_spanish)
Config: Voice Assistant Settings single doctype (enabled, api_key, model, endpoint_url), falling back to site_config.voice_assistant_settings. Default model llama-3.3-70b-specdec on Groq, 15s timeout.

The 23 registered tools
READ-ONLY — 20 tools (every one gated by frappe.has_permission(..., "read") and throws PermissionError on failure):

Tool	Reads	Permission gate
handle_sales	Sales Invoice	✅ read
handle_purchase	Purchase Invoice	✅ read
handle_profit	Sales + Purchase Invoice	✅ both
handle_outstanding	Sales Invoice	✅ read
handle_stock_balance	Bin + Item	✅ both
handle_low_stock	Bin + Item	✅ both
handle_expense	Expense Claim	✅ read
handle_payment_received	Payment Entry	✅ read
handle_top_customers	Sales Invoice	✅ read
handle_top_products	Sales Invoice	✅ read
handle_pending_orders	Sales/Purchase Order	✅ read
handle_new_customers	Customer	✅ read
handle_payroll	Salary Slip	✅ read
handle_count	any doctype	✅ read (per-doctype, line 1739)
handle_leave_balance	Leave Allocation	✅ read
handle_employees_on_leave	Leave Application	✅ read
handle_employee_directory	Employee	✅ read
handle_appraisals	Appraisal	✅ read
handle_lms_summary	Course, Program Enrollment, Course Enrollment	✅ read
handle_verifactu_status	Sales Invoice (custom_verifactu_status)	✅ read
NAVIGATION — 1 tool: navigate_to (doctype / report / page) — returns a client-side action, no permission check (Frappe's own list view enforces it downstream).

WRITE — 2 tools:

Tool	Reality
handle_create_record	Hard-coded allowlist: ToDo, Lead, Task, Comment only. Anything else → frappe.throw(PermissionError). Then frappe.has_permission(doctype, "create"). Uses doc.insert() so framework validation applies.
handle_fill_form	Returns {action: "fill_form", form_values} to the browser, which populates the currently open form. Never persists. No permission check. No field validation. The user still has to hit Save.
The education/training coverage today — this is the gap
One handler, handle_lms_summary, and it does three frappe.db.count() calls: total Courses, total Program Enrollments, total Course Enrollments. No filtering, no student lookup, no attendance, no certification, nothing ATU-specific.

Navigation-only (in DOCTYPE_MAP, can be opened but not queried): Course, Program Enrollment, Course Enrollment, Incidencia Estudiante, Certificados Centros de Formación, Planes Formativos, Especialidades Formativas.

Note: "course enrollment" and "matriculación de curso" both map to Program Enrollment at voice_command.py:2108 — that's a bug; "matriculación de curso" should map to Course Enrollment.

Completely absent: Student, Student Applicant, Student Attendance, Aula, Program, Oferta Convocada, Solicitud Formacion Empresa, Instructor, Assessment Result, Encuesta de Calidad, Imparticion de Certificado, and every justificacionatu doctype.

search_entity() autocomplete supports only: item, warehouse, customer, supplier, employee. No student, no course, no instructor.

Two architectural blockers for "agentic"
These are the real reasons it can't do work today, independent of which tools you add:

1. Single tool call, no loop. Line ~257: tool_call = tool_calls[0]. Exactly one tool is executed per utterance, and its result is returned straight to the user — never fed back to the model. So "enrol María in the Excel course and mark her present today" is structurally impossible: it needs resolve-student → resolve-course → create-enrollment → create-attendance.

2. No conversation memory. The messages array is rebuilt every call as [system, user]. There is no history, so no clarification turns ("which María?"), no confirmations ("shall I proceed?"), and context only carries the currently-open form's doctype/docname.

There's also no audit trail — write actions aren't logged anywhere beyond the created document itself.

Part 5 — What agentic requires
To make it act on the training layer, in dependency order:

Agent loop — replace the single-shot call with a while loop (cap ~6 iterations): execute tool → append {role: "tool", tool_call_id, content} → re-call Groq → repeat until the model returns prose. This is the single highest-value change, and it unlocks everything else.
Session memory — persist message history per user+session (a Voice Assistant Conversation doctype or Redis), so multi-turn clarification works.
Confirmation gate — classify tools as read / write / destructive. Write tools return a {action: "confirm", preview, token} first; the browser shows a diff, the user confirms, and the token is redeemed. Essential when you're touching funded-training records.
Entity resolution for the domain — extend search_entity with student, course, program, instructor, aula, plan, and add _verify_or_fuzzy_match types for them. Spanish names need accent-insensitive matching.
Replace the allowlist with a permission-driven policy. Drop the hard-coded four doctypes; drive writes off frappe.has_permission(dt, "create"/"write") plus an explicit per-doctype tool definition (so the model can only touch doctypes you've deliberately exposed).
Audit log — a Voice Assistant Action Log recording user, utterance, tool, args, result, and confirmation token for every write. You will want this the first time an auditor asks who changed an attendance record.
Concrete training-layer tools worth building, roughly in value order:

Tool	Type	Notes
get_student_profile(student)	read	Reuse students_api.get_student_details — already returns applicant data, enrolments, courses, files
get_student_attendance(student, course?)	read	Reuse get_resumen_asistencia_alumno
get_course_roster(course)	read	Reuse get_alumnos_curso_asistencia
get_attendance_alerts(course?)	read	Reuse get_alertas_asistencia — drop-out risk
get_plan_status(plan)	read	estado, subvención concedida vs ejecutada, seats filled vs alumnado_concedido
get_course_status(course)	read	state, dates, milestone %, hours, enrolled vs cupo
mark_attendance(course, student, date, status)	write	Wrap guardar_asistencia — highest-frequency daily action, ideal first write
enrol_student(student, course)	write	Must respect cupo_af seat cap
create_student_applicant(fields)	write	Validation-heavy (postcode↔province, FUNDAE fields)
update_applicant_status(applicant, status)	write	Applied→Approved→Admitted
create_student_incident(student, ...)	write	Low risk, good second write
send_course_diplomas(course)	destructive	Wraps generar_diplomas — emails real people; always confirm
Start with mark_attendance behind the confirmation gate. It's the most-repeated action in the business, it's low blast-radius, and it exercises the full agent loop (resolve student → resolve course → resolve date → write) without risking a funding dossier.

Want me to implement the agent loop and the first batch of read tools, or write this up as a shareable spec page first?
