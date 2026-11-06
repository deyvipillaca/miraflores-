# Enfoque arquitectónico

## Sistema Web de Gestión de Matrícula y Asistencia para el Colegio Miraflores

| Elemento | Descripción aplicada al Sistema de Matrícula y Asistencia |
|---|---|
| **Patrón / enfoque arquitectónico** | Clean Architecture (Arquitectura Limpia), aplicada dentro de cada módulo del monolito modular. |
| **Objetivo** | Separar responsabilidades y controlar las dependencias hacia el dominio, de modo que las reglas de matrícula y asistencia no dependan de la interfaz web ni de tecnologías externas. |
| **¿Qué problema resuelve?** | Evita el acoplamiento entre la interfaz web (portal del apoderado, panel de secretaría, registro de asistencia), las reglas de negocio (matrícula única por año, control de vacantes, estados de matrícula, tipos de asistencia, plazos de justificación) y las tecnologías externas, como la base de datos, el servicio de notificaciones (correo / SMS / WhatsApp), el almacenamiento de documentos y la exportación a SIAGIE. |
| **Capas definidas** | Presentación, Aplicación, Dominio e Infraestructura. |
| **Beneficios** | <ul><li>Facilita el mantenimiento y las pruebas unitarias de las reglas de matrícula y asistencia sin depender de servicios externos.</li><li>Permite cambiar implementaciones técnicas (proveedor de mensajería, almacenamiento, motor de base de datos) sin modificar innecesariamente las reglas del negocio.</li><li>Mejora la organización y separación de responsabilidades del código.</li><li>Favorece la disponibilidad: si falla un servicio externo, el núcleo del sistema sigue funcionando.</li></ul> |

## Capas definidas

| Capa | Responsabilidad en el sistema | Componentes |
|---|---|---|
| **Presentación** | Recibe las peticiones del cliente web, autentica, valida la entrada y responde en JSON. Muestra formularios de matrícula, listas de asistencia y reportes. | Rutas y controladores (`*.routes.js`, `*.controller.js`), portal del apoderado, panel de secretaría, registro de asistencia. |
| **Aplicación** | Coordina los casos de uso: registrar matrícula, validar documentos, confirmar matrícula, registrar asistencia, resolver justificaciones, enviar notificaciones y generar reportes. | Servicios de aplicación (`*.service.js`), casos de uso, puertos (interfaces) hacia la infraestructura. |
| **Dominio** | Contiene las reglas de negocio y las entidades, sin depender de ninguna tecnología. | Estudiante, Apoderado, Periodo escolar, Grado, Sección, Matrícula, Documento, Pago, Asistencia, Justificación, Notificación, Usuario; reglas de estados y de vacantes. |
| **Infraestructura** | Implementa los detalles técnicos y la comunicación con sistemas externos. | Repositorios (Sequelize + PostgreSQL), proveedor de notificaciones, almacenamiento de documentos, exportador SIAGIE. |

## Regla de dependencias

```
Presentación ──► Aplicación ──► Dominio ◄── Infraestructura
```

- Las dependencias apuntan siempre **hacia el dominio**.
- El dominio no conoce a Express, Sequelize, PostgreSQL ni a ningún servicio externo.
- La infraestructura implementa las interfaces (puertos) que define la capa de aplicación (por ejemplo, `NotificadorPort`, `MatriculaRepositoryPort`, `AcademicoPort`).

## Relación con el estilo arquitectónico

El **estilo** (monolito modular en 3 capas físicas) y el **enfoque** (Clean Architecture en 4 capas lógicas) se complementan:

| Capa física del monolito (estilo) | Capas de Clean Architecture (enfoque) |
|---|---|
| 1. Presentación (`routes`, `controller`) | Presentación |
| 2. Lógica de negocio (`service`) | Aplicación + Dominio |
| 3. Datos (`repository`, Sequelize) | Infraestructura |

## Ejemplo: flujo "Registrar una matrícula"

1. **Presentación:** `matricula.controller` recibe la petición de secretaría o del apoderado y valida los datos de entrada.
2. **Aplicación:** el caso de uso `RegistrarMatricula` verifica permisos, el periodo habilitado y coordina el proceso con los módulos `academico`, `documentos` y `pagos`.
3. **Dominio:** la entidad `Matricula` valida sus reglas (estudiante, tipo y estado válidos, documentos completos y pago registrado); el módulo `academico` reserva la vacante de la sección y la matrícula queda *matriculada*.
4. **Infraestructura:** `matricula.repository` guarda la matrícula en PostgreSQL dentro de una transacción y el proveedor de notificaciones avisa al apoderado.
