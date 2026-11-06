# Decisiones arquitectónicas

## Sistema Web de Gestión de Matrícula y Asistencia para el Colegio Miraflores

### Resumen

| ID | Decisión | Drivers relacionados |
|---|---|---|
| DEC01 | Monolito modular en lugar de microservicios | DA01, DA07, DA08 |
| DEC02 | Arquitectura en tres capas dentro de cada módulo | DA04, DA08 |
| DEC03 | Node.js 20 LTS + Express como plataforma backend | DA02, DA08 |
| DEC04 | API REST con JSON sobre HTTPS | DA05, DA06 |
| DEC05 | PostgreSQL como única base de datos | DA02, DA04 |
| DEC06 | Sequelize como ORM y patrón Repository | DA08 |
| DEC07 | Autenticación JWT y control de acceso por roles | DA03 |
| DEC08 | Comunicación entre módulos solo mediante services | DA08 |
| DEC09 | Integraciones externas encapsuladas (notificaciones, almacenamiento y SIAGIE) | DA05, DA01 |
| DEC10 | Control de vacantes y matrícula única mediante transacciones | DA02, DA04 |
| DEC11 | Cliente web responsive y registro de asistencia ágil | DA06, DA01 |
| DEC12 | Flujo de estados de la matrícula | DA04 |
| DEC13 | Bitácora de auditoría de matrículas y asistencias | DA03, DA04 |
| DEC14 | Diez módulos de negocio con responsabilidad única | DA07, DA08 |
| DEC15 | Pagos registrados por Tesorería, sin pasarela en la primera versión | DA04, DA05 |

---

### DEC01. Monolito modular en lugar de microservicios
- **Contexto:** el colegio cuenta con un equipo técnico reducido y recursos limitados, y el sistema debe estar disponible en campañas de matrícula y en la jornada escolar.
- **Decisión:** construir una sola aplicación desplegable, dividida en 10 módulos de negocio (usuarios, estudiantes, academico, matricula, documentos, pagos, asistencia, justificaciones, notificaciones, reportes).
- **Justificación:** simplifica el despliegue, la operación y el monitoreo; evita la complejidad y la latencia de red de los microservicios.
- **Consecuencias:** (+) menor costo y complejidad; (−) todo se escala junto. Los límites de módulo permiten extraer servicios en el futuro.

### DEC02. Arquitectura en tres capas
- **Decisión:** organizar cada módulo en presentación (routes/controller), lógica de negocio (service) y datos (repository).
- **Justificación:** separa responsabilidades y facilita pruebas y cambios.
- **Consecuencias:** cada capa solo invoca a la inmediatamente inferior; evita lógica de negocio en controladores y SQL en servicios.

### DEC03. Node.js 20 LTS + Express
- **Decisión:** usar Node.js 20 LTS con Express como framework web.
- **Justificación:** ecosistema amplio, buen manejo de E/S concurrente (muchos apoderados y docentes conectados a la vez), versión con soporte a largo plazo.
- **Consecuencias:** un solo lenguaje (JavaScript) en cliente y servidor.

### DEC04. API REST con JSON sobre HTTPS
- **Decisión:** exponer la API en `/api/v1/` con JSON sobre HTTPS.
- **Justificación:** estándar, simple de consumir por el cliente web y por una futura app móvil; el versionado protege a los clientes de cambios.
- **Consecuencias:** el cifrado en tránsito protege los datos personales de estudiantes y apoderados.

### DEC05. PostgreSQL como única base de datos
- **Decisión:** una única base de datos relacional PostgreSQL.
- **Justificación:** el dominio es fuertemente relacional (estudiante–apoderado, matrícula–sección, asistencia–matrícula) y requiere transacciones ACID y restricciones únicas para garantizar la integridad.
- **Consecuencias:** una sola fuente de verdad; requiere respaldos periódicos y, si es posible, réplica para disponibilidad.

### DEC06. Sequelize (ORM) y patrón Repository
- **Decisión:** el acceso a datos pasa por repositories que usan Sequelize, con modelos y pool de conexiones compartidos (`src/shared/db`).
- **Justificación:** aísla la lógica de negocio del motor de base de datos y centraliza las consultas.
- **Consecuencias:** facilita cambios de esquema y pruebas con repositorios simulados.

### DEC07. Autenticación JWT y control de roles
- **Decisión:** autenticar con tokens JWT y autorizar por rol (Administrador, Secretaría, Tesorería, Director, Docente, Auxiliar, Apoderado, Estudiante) mediante middleware.
- **Justificación:** el sistema maneja datos de menores; cada usuario solo debe ver y modificar lo que le corresponde (un apoderado solo ve a sus hijos, un docente solo sus secciones).
- **Consecuencias:** además del rol, se valida la pertenencia del recurso (por ejemplo, que la sección pertenezca al docente).

### DEC08. Comunicación entre módulos solo mediante services
- **Decisión:** un módulo no accede al repository ni a las tablas de otro módulo; usa su service.
- **Justificación:** mantiene bajo acoplamiento y permite extraer un módulo a futuro.
- **Consecuencias:** requiere disciplina de diseño y revisiones de código.

### DEC09. Integraciones externas encapsuladas
- **Decisión:** el envío de mensajes, el almacenamiento de documentos y la exportación a SIAGIE se encapsulan en adaptadores detrás de puertos (interfaces).
- **Justificación:** los servicios externos pueden fallar o cambiar de proveedor; aislarlos permite aplicar reintentos, tiempos de espera y reemplazos.
- **Consecuencias:** si el servicio de notificaciones falla, la matrícula y la asistencia se registran igual y el aviso se reintenta después.

### DEC10. Control de vacantes y matrícula única mediante transacciones
- **Contexto:** durante la campaña de matrícula, varios usuarios pueden pedir la misma última vacante al mismo tiempo.
- **Decisión:** reservar la vacante (módulo `academico`) y crear la matrícula en una sola transacción con bloqueo de la fila de la sección, y garantizar la matrícula única por estudiante y periodo con una restricción única en la base de datos.
- **Justificación:** evita sobrecupo y matrículas duplicadas aun con concurrencia.
- **Consecuencias:** el segundo intento recibe un error claro (sin vacante o ya matriculado); se añade una pequeña espera por el bloqueo.

### DEC11. Cliente web responsive y registro de asistencia ágil
- **Decisión:** cliente web (HTML/CSS/JS) responsive, con la lista de asistencia marcada por defecto como "presente" para registrar solo las excepciones.
- **Justificación:** los docentes toman asistencia en pocos minutos y muchos apoderados usan el celular.
- **Consecuencias:** se puede evolucionar a PWA para guardar la asistencia sin conexión y sincronizarla después.

### DEC12. Flujo de estados de la matrícula
- **Decisión:** la matrícula sigue los estados solicitada → en revisión → matriculada, con salidas posibles a rechazada, retirada o anulada.
- **Justificación:** la vacante solo se consume al confirmar la matrícula y los documentos deben validarse antes.
- **Consecuencias:** añade un paso de revisión de secretaría, pero ordena el proceso y deja trazabilidad.

### DEC13. Bitácora de auditoría
- **Decisión:** un middleware registra quién, cuándo y qué cambió en matrículas, asistencias y justificaciones.
- **Justificación:** las correcciones de asistencia y los cambios de matrícula deben ser trazables y revisables.
- **Consecuencias:** mayor volumen de datos; se define una política de retención.

### DEC14. Diez módulos de negocio con responsabilidad única
- **Decisión:** separar la estructura académica (`academico`), los archivos (`documentos`), los pagos (`pagos`) y las justificaciones (`justificaciones`) de los módulos `matricula` y `asistencia`.
- **Justificación:** cada módulo cambia por razones distintas (reglas académicas, requisitos documentarios, tesorería, reglamento de asistencia) y puede evolucionar sin afectar a los demás; `documentos` se reutiliza para matrícula y justificaciones.
- **Consecuencias:** más puntos de colaboración entre módulos, que deben pasar siempre por sus services (DEC08).

### DEC15. Pagos registrados por Tesorería, sin pasarela en la primera versión
- **Decisión:** el módulo `pagos` registra pagos de matrícula a partir del comprobante o depósito verificado por Tesorería, y expone su estado a `matricula`.
- **Justificación:** reduce la complejidad inicial y los costos de integración; el puerto `PagosPort` permite incorporar una pasarela de pago más adelante sin cambiar el dominio.
- **Consecuencias:** la verificación del pago es manual en esta etapa.
