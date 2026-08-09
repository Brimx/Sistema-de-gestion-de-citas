# Kairos — Sistema de Gestión de Citas Médicas

Aplicación de escritorio desarrollada en **Java + JavaFX** con interfaz **Material Design** vía CSS (sin librerías externas) para la gestión integral de citas médicas. Permite a pacientes agendar citas, a médicos gestionar su agenda del día y a administradores supervisar el sistema completo.

![Java](https://img.shields.io/badge/Java-17+-orange?logo=java) ![JavaFX](https://img.shields.io/badge/JavaFX-21+-blue) ![Paradigma](https://img.shields.io/badge/Paradigma-POO-green) ![SQLite](https://img.shields.io/badge/Base%20de%20datos-SQLite-lightgrey?logo=sqlite)

---

## Tabla de contenidos

- [Descripción general](#descripción-general)
- [Tecnologías utilizadas](#tecnologías-utilizadas)
- [Arquitectura del proyecto](#arquitectura-del-proyecto)
- [Estructura de paquetes](#estructura-de-paquetes)
- [Modelos de datos](#modelos-de-datos)
- [Flujo de una cita](#flujo-de-una-cita)
- [Reglas de validación](#reglas-de-validación)
- [Roles del sistema](#roles-del-sistema)
- [Cómo ejecutar el proyecto](#cómo-ejecutar-el-proyecto)
- [Base de datos](#base-de-datos)
- [Mejoras futuras](#mejoras-futuras)

---

## Descripción general

El sistema gestiona el ciclo de vida completo de una cita médica: desde que el paciente la solicita hasta que el médico la atiende. Contempla tres tipos de usuario (paciente, médico, administrador), con dashboards separados por rol y autenticación real contra SQLite.

---

## Tecnologías utilizadas

| Tecnología | Versión | Uso |
|---|---|---|
| Java | 17+ | Lenguaje principal |
| JavaFX | 21.0.2 | Interfaz gráfica de usuario |
| FXML | — | Definición declarativa de vistas |
| SQLite | — | Base de datos embebida |
| Xerial SQLite JDBC | 3.45.2.0 | Conector Java ↔ SQLite |
| CalendarFX | 11.12.7 | Componentes de calendario |
| IBM Plex Sans | — | Tipografía principal del sistema |
| CSS (Material Design / Indigo) | — | UI moderna con paleta índigo, sombras y elevaciones |
| Maven | — | Gestión de dependencias y build |

---

## Arquitectura del proyecto

El proyecto sigue una **arquitectura en capas** (Layered Architecture) combinada con el patrón **MVC**:

```
┌─────────────────────────────────────┐
│           VISTA (View)              │  JavaFX FXML + ViewManager
├─────────────────────────────────────┤
│        CONTROLADOR (Controller)     │  LoginController, RegistroController,
│                                     │  DashboardPacienteController,
│                                     │  DashboardMedicoController,
│                                     │  DashboardAdminController
├─────────────────────────────────────┤
│          SERVICIO (Service)         │  CitaService, PacienteService, Session
├─────────────────────────────────────┤
│            DAO (Data Access)        │  UsuarioDAO, CitaDAO, PacienteDAO,
│                                     │  MedicoDAO, AdministradorDAO
├─────────────────────────────────────┤
│          BASE DE DATOS              │  SQLite (citasmedicas.db)
└─────────────────────────────────────┘
        ↕ todas las capas usan ↕
┌─────────────────────────────────────┐
│       MODELOS + ENUMS               │  Cita, Paciente, Medico, Turno...
└─────────────────────────────────────┘
```

Cada capa solo se comunica con la inmediatamente inferior, nunca salta capas.

---

## Estructura de paquetes

```
co.edu.upc.citasmedicas/
│
├── Launcher.java              ← Punto de entrada compatible con exec-maven-plugin
├── Main.java                  ← Clase principal de JavaFX (extiende Application)
│
├── controller/                ← Controlan la lógica de cada pantalla FXML
│   ├── LoginController.java
│   ├── RegistroController.java
│   ├── DashboardPacienteController.java
│   ├── DashboardMedicoController.java
│   └── DashboardAdminController.java
│
├── dao/                       ← Acceso a base de datos (operaciones CRUD)
│   ├── DatabaseConnection.java   (inicializa BD y datos demo)
│   ├── UsuarioDAO.java           (autenticación y guardado multi-rol)
│   ├── PacienteDAO.java
│   ├── MedicoDAO.java
│   ├── AdministradorDAO.java
│   ├── CitaDAO.java
│   ├── AgendaMedicaDAO.java      (horarios del médico)
│   ├── BloqueoAgendaDAO.java     (bloqueos temporales)
│   ├── HistorialClinicoDAO.java  (historial por cita completada)
│   └── ConfiguracionDAO.java     (clave/valor — código admin, etc.)
│
├── enums/                     ← Constantes tipadas del dominio
│   ├── Rol.java               (PACIENTE, MEDICO, ADMIN)
│   ├── EstadoCita.java        (PENDIENTE, CONFIRMADA, COMPLETADA, CANCELADA, NO_ASISTIO)
│   ├── Especialidad.java      (17 especialidades médicas)
│   ├── ServicioCita.java      (17 servicios con duración dinámica y grupo)
│   ├── TipoCita.java          (PRESENCIAL, VIRTUAL)
│   └── OrigenCita.java        (PACIENTE, CONTROL)
│
├── model/                     ← Entidades del dominio
│   ├── Usuario.java           (abstracta — base de todos los usuarios)
│   ├── Paciente.java
│   ├── Medico.java
│   ├── Administrador.java
│   ├── Cita.java              (entidad central — duración dinámica, origen, sobrecupo)
│   ├── AgendaMedica.java      (horarios del médico por día de semana)
│   ├── BloqueoAgenda.java     (bloqueos temporales por fecha)
│   └── HistorialClinico.java  (diagnóstico, receta, remisión por cita)
│
├── service/                   ← Reglas de negocio y validaciones
│   ├── CitaService.java       (agendar, sobrecupo, inasistencias, completar)
│   ├── DisponibilidadService.java (cálculo de slots dinámicos)
│   ├── InasistenciaService.java   (singleton — auto-detección cada 5 min)
│   └── Session.java           (sesión del usuario autenticado)
│
└── view/
    └── ViewManager.java       ← Carga y muestra vistas FXML
```

---

## Modelos de datos

### Jerarquía de usuarios

```
Usuario (abstracta)
├── Administrador   → gestiona médicos, pacientes y reportes
├── Medico          → gestiona su agenda del día
└── Paciente        → solicita y consulta sus citas
```

`Usuario` es una clase abstracta que obliga a cada rol a implementar `getMenuOpciones()`. Almacena datos comunes: id, nombre, apellido, email, password, teléfono, rol y estado activo.

Además de los campos heredados, cada rol extiende con datos específicos:

- **Paciente**: tipoDocumento, numeroDocumento, fechaNacimiento, direccion, eps
- **Medico**: registroMedico, especialidad, tipoDocumento, numeroDocumento, fechaNacimiento, direccion, eps
- **Administrador**: codigoAdmin, cargo, tipoDocumento, numeroDocumento, fechaNacimiento, direccion, eps

El registro de nuevos usuarios usa un **asistente multi-paso** (`registro.fxml`) donde:
1. **Paso 1**: se selecciona nombre, apellido y rol (con código de admin si aplica).
2. **Paso 2**: se completan los datos específicos del rol (email, password, teléfono, documentos, fecha de nacimiento, dirección, EPS, y especialidad/registro si es médico).

El código de administrador se valida contra la tabla `configuracion` en la base de datos (en lugar de un valor fijo en código), permitiendo cambiarlo desde el panel de administración.

### Entidades principales

**`Cita`** — Entidad central del sistema. Conecta un `Paciente` con un `Medico`.
- Nace siempre en estado `PENDIENTE`
- Duración dinámica: depende del servicio (`ServicioCita.duracionMinutos`) y del origen (control usa `duracionControlMinutos`)
- Soporta sobrecupo (`sobrecupo = true`) que salta validación de disponibilidad
- Origen: `PACIENTE` (agendada por paciente) o `CONTROL` (agendada por médico como control)
- Nunca se borra físicamente (borrado lógico mediante cambio de estado)

**`AgendaMedica`** — Define los horarios del médico por día de semana.
- Campos: médico, día de semana (1=lunes…7=domingo), hora inicio, hora fin, slot en minutos
- Los slots disponibles se generan dinámicamente según el `slotMinutos` y la duración del servicio

**`BloqueoAgenda`** — Bloqueos temporales sobre la agenda de un médico.
- Por fecha específica, con rango horario opcional (si es null, bloquea todo el día)
- Inhabilita parcial o totalmente la disponibilidad del médico en esa fecha

**`HistorialClinico`** — Registro clínico asociado a una cita completada.
- Campos: diagnóstico, enfermedad actual, receta, remisión, notas
- Se guarda al finalizar la consulta y la cita pasa a estado `COMPLETADA`

### Esquema de base de datos

```sql
CREATE TABLE usuarios (
    id TEXT PRIMARY KEY,
    nombre TEXT NOT NULL,
    apellido TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL,
    telefono TEXT NOT NULL,
    rol TEXT NOT NULL,
    activo INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE pacientes (
    usuario_id TEXT PRIMARY KEY,
    tipo_documento TEXT NOT NULL,
    numero_documento TEXT NOT NULL,
    fecha_nacimiento TEXT NOT NULL,
    direccion TEXT NOT NULL,
    eps TEXT NOT NULL,
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE TABLE medicos (
    usuario_id TEXT PRIMARY KEY,
    registro_medico TEXT NOT NULL,
    especialidad TEXT NOT NULL,
    tipo_documento TEXT NOT NULL DEFAULT '',
    numero_documento TEXT NOT NULL DEFAULT '',
    fecha_nacimiento TEXT,
    direccion TEXT DEFAULT '',
    eps TEXT DEFAULT '',
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE TABLE administradores (
    usuario_id TEXT PRIMARY KEY,
    codigo_admin TEXT NOT NULL,
    cargo TEXT NOT NULL DEFAULT '',
    tipo_documento TEXT NOT NULL DEFAULT '',
    numero_documento TEXT NOT NULL DEFAULT '',
    fecha_nacimiento TEXT,
    direccion TEXT DEFAULT '',
    eps TEXT DEFAULT '',
    FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

CREATE TABLE configuracion (
    clave TEXT PRIMARY KEY,
    valor TEXT NOT NULL
);

CREATE TABLE citas (
    id TEXT PRIMARY KEY,
    paciente_id TEXT NOT NULL,
    medico_id TEXT NOT NULL,
    especialidad TEXT NOT NULL,
    servicio TEXT NOT NULL DEFAULT 'MEDICINA_GENERAL',
    fecha TEXT NOT NULL,
    hora_inicio TEXT NOT NULL,
    duracion INTEGER NOT NULL DEFAULT 20,
    estado TEXT NOT NULL,
    tipo TEXT NOT NULL,
    motivo TEXT NOT NULL,
    origen TEXT NOT NULL DEFAULT 'PACIENTE',
    sobrecupo INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (paciente_id) REFERENCES pacientes(usuario_id),
    FOREIGN KEY (medico_id) REFERENCES medicos(usuario_id)
);

CREATE TABLE agenda_medica (
    id TEXT PRIMARY KEY,
    medico_id TEXT NOT NULL,
    dia_semana INTEGER NOT NULL,
    hora_inicio TEXT NOT NULL,
    hora_fin TEXT NOT NULL,
    slot_minutos INTEGER NOT NULL DEFAULT 20,
    FOREIGN KEY (medico_id) REFERENCES medicos(usuario_id)
);

CREATE TABLE bloqueos_agenda (
    id TEXT PRIMARY KEY,
    medico_id TEXT NOT NULL,
    fecha TEXT NOT NULL,
    hora_inicio TEXT,
    hora_fin TEXT,
    motivo TEXT NOT NULL,
    FOREIGN KEY (medico_id) REFERENCES medicos(usuario_id)
);

CREATE TABLE historial_clinico (
    id TEXT PRIMARY KEY,
    cita_id TEXT NOT NULL,
    medico_id TEXT NOT NULL,
    paciente_id TEXT NOT NULL,
    fecha_consulta TEXT NOT NULL,
    diagnostico TEXT,
    enfermedad_actual TEXT,
    receta TEXT,
    remision TEXT,
    notas TEXT,
    FOREIGN KEY (cita_id) REFERENCES citas(id),
    FOREIGN KEY (medico_id) REFERENCES medicos(usuario_id),
    FOREIGN KEY (paciente_id) REFERENCES pacientes(usuario_id)
);
```

---

## Flujo de una cita

```
[Paciente / Admin solicita cita]     [Médico agenda control]
        │                                     │
        ▼                                     │
  Estado: PENDIENTE ◄─────────────────────────┘ (origen=CONTROL)
        │
        ▼ (admin confirma o queda automática)
  Estado: CONFIRMADA ──────────────► CANCELADA
        │
        ├──► (médico completa consulta + historial clínico)
        │     Estado: COMPLETADA
        │
        └──► (auto-detección cada 5 min)
              Estado: NO_ASISTIO
```

---

## Reglas de validación

- **Disponibilidad dinámica**: los slots se generan desde `agenda_medica` según `slot_minutos` y se filtran restando citas ocupadas + bloqueos activos.
- **Duración variable**: cada servicio define su propia duración (`duracionMinutos` para primera vez, `duracionControlMinutos` para controles).
- **Frecuencia por especialidad**: un paciente no puede tener dos citas activas (`PENDIENTE` o `CONFIRMADA`) en la misma especialidad.
- **Sobrecupo**: salta la validación de disponibilidad. Se marca visualmente con borde rojo izquierdo en la tabla.
- **Bloqueos de agenda**: inhabilitan slots completos o parciales para un médico en una fecha específica.
- **Auto-detección de inasistencias**: al cargar cada dashboard y luego cada 5 minutos vía `ScheduledExecutorService`, las citas vencidas pasan a `NO_ASISTIO`.
- **Origen de cita**: las agendadas por el paciente son `PACIENTE`; las agendadas por el médico como control son `CONTROL`.
- **Historial clínico**: al completar una consulta, el médico llena el formulario (diagnóstico obligatorio) y se persiste en `historial_clinico`.
- **Email duplicado**: no se permite registrar dos usuarios con el mismo email.
- **Formato de nombre/apellido**: solo letras (incluyendo tildes y ñ) y espacios — números y caracteres especiales son rechazados via `ValidacionService.mensajeErrorNombre()`.
- **Formato de documento**: solo dígitos via `ValidacionService.mensajeErrorNumeroDocumento()`.
- **Formato de EPS**: solo letras y espacios via `ValidacionService.mensajeErrorEps()`.
- **Tipo de documento**: ComboBox fijo (CC, TI, CE, Pasaporte) en todos los formularios — no se permiten valores arbitrarios.
- **Email en modify dialogs**: ahora se valida formato en los diálogos de modificar datos de paciente, médico y administrador.
- **Null safety**: se corrigió `NullPointerException` en `PacienteDAO.actualizarTodo()` cuando la fecha de nacimiento es null.
- **Borrado lógico**: las citas canceladas y usuarios desactivados permanecen en la BD.

---

## Roles del sistema

### Paciente
- Registrarse en el sistema (asistente multi-paso con datos personales)
- Solicitar cita médica (seleccionando médico, fecha, hora, tipo y motivo)
- El motivo de consulta usa un **TextArea expandible** con mínimo 3 líneas, scroll automático y botón para vista completa en diálogo modal
- Ver mis citas activas
- Cancelar una cita (excepto si ya fue completada)

### Médico
- Ver agenda del día con citas filtradas por estado
- **Agendar control**: agenda cita de control para un paciente (origen=CONTROL, usa duración reducida)
- **Iniciar consulta**: abre formulario de historial clínico (diagnóstico, receta, remisión) y completa la cita
- Marcar paciente como **no asistió**
- **Reprogramar cita**: cambia fecha/hora de una cita existente
- **Gestionar horarios**: administra su propia agenda médica (días, horarios, slots)

### Administrador
- Gestionar pacientes (listar, editar, desactivar)
- Gestionar médicos (listar, registrar nuevo)
- Ver todas las citas del sistema
- **Agregar / modificar cita** a cualquier paciente con selector de tipo (primera vez / control) y sobrecupo
- **Gestionar bloqueos**: bloquea parcial o totalmente la agenda de un médico en una fecha
- **Gestionar horarios**: administra la agenda médica de cualquier médico
- Marcar paciente como **no asistió**

---

## Cómo ejecutar el proyecto

### Requisitos previos
- Java 17 o superior
- Maven 3.8+
- NetBeans 17+ (recomendado) o cualquier IDE con soporte Maven

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/Brimx/Sistema-de-gestion-de-citas.git
cd Sistema-de-gestion-de-citas

# 2. Compilar el proyecto
mvn clean compile

# 3. Ejecutar
mvn javafx:run
```

> Si usas NetBeans, abre el proyecto y presiona **Run** (F6). La primera ejecución crea automáticamente la base de datos SQLite con datos de demostración.

### Credenciales de demostración

| Rol | Email | Contraseña |
|-----|-------|------------|
| Paciente | `paciente@demo.com` | `1234` |
| Paciente | `pedro@demo.com` | `1234` |
| Paciente | `ana@demo.com` | `1234` |
| Paciente | `sofia@demo.com` | `1234` |
| Paciente | `diego@demo.com` | `1234` |
| Paciente | `carolina@demo.com` | `1234` |
| Paciente | `valentina@demo.com` | `1234` |
| Paciente | `felipe@demo.com` | `1234` |
| Paciente | `camila@demo.com` | `1234` |
| Médico (Medicina General) | `medico@demo.com` | `1234` |
| Médico (Odontología) | `odontologia@demo.com` | `1234` |
| Médico (Pediatría) | `pediatria@demo.com` | `1234` |
| Médico (Dermatología) | `dermatologia@demo.com` | `1234` |
| Médico (Psicología) | `psicologia@demo.com` | `1234` |
| Médico (Psiquiatría) | `psiquiatria@demo.com` | `1234` |
| Médico (Nutrición) | `nutricion@demo.com` | `1234` |
| Administrador | `admin@demo.com` | `1234` |

---

## Base de datos

El sistema usa **SQLite** como base de datos embebida. El archivo `citasmedicas.db` se crea automáticamente en el directorio raíz del proyecto al ejecutar por primera vez.

**Ventajas de SQLite:**
- No requiere instalación de servidor
- Archivo único portátil
- Transacciones ACID
- `PRAGMA foreign_keys = ON` para integridad referencial

La conexión se gestiona a través de `DatabaseConnection.java` que también inicializa las tablas y los datos demo automáticamente en la primera ejecución.

---

## Cambios recientes

### Junio 2026 — Validaciones de formato y correcciones de robustez

- **ValidacionService extendido**: nuevos métodos `mensajeErrorNombre()`, `mensajeErrorNumeroDocumento()` y `mensajeErrorEps()` con regex para rechazar entradas inválidas en tiempo de envío.
- **tipoDoc unificado**: todos los modify dialogs ahora usan `ComboBox<String>` con opciones fijas (CC, TI, CE, Pasaporte) igual que el registro.
- **Email validado en modify**: los diálogos de modificar datos de Paciente, Médico y Administrador ahora validan el formato del email antes de guardar.
- **Null safety**: corregido `NullPointerException` en `PacienteDAO.actualizarTodo()` y `UsuarioDAO.guardarPaciente()` cuando `fechaNacimiento` es null.
- **Documentación**: agregada sección 12 (Banco de Preguntas para Defensa) en `docs/ARQUITECTURA.md` con 30 preguntas y respuestas técnicas.
- **Demo expandido**: se agregaron 3 pacientes nuevos (Valentina, Felipe, Camila), 2 médicos nuevos (Psiquiatría y Nutrición), 10 citas adicionales, 5 historiales clínicos, 2 bloqueos de agenda y 8 registros de agenda. Total: 9 pacientes, 7 médicos, 23 citas demo.
- **Reprogramación automática**: al eliminar un médico, `CitaService.reprogramarCitasDelMedico()` reasigna sus citas futuras a otros médicos de la misma especialidad. Busca el slot más cercano (mismo día o hasta 14 días después) o cancela la cita si no hay disponibilidad. Integrado en el dashboard del administrador.

---

## Mejoras futuras

- **Kanban / Sala de Espera**: vista del médico con columnas de estado tipo kanban para mover pacientes de "En espera" a "En consulta" a "Atendido".
- **Reportes**: generar estadísticas de citas por médico, especialidad y periodo.
- **Supabase**: migrar a base de datos en la nube para acceso multi-sede y sincronización en tiempo real.

---

## Autores

Proyecto académico — **Programación Orientada a Objetos**  
Universidad Popular del Cesar · Facultad de Ingenierías y Tecnológicas

| Estudiante | Santiago Andres Montenegro Muñoz |
