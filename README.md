# Checkpoint 4 · Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Alumno:** Test CH Carlos · **Caso:** Electrónica Cobra · **n8n:** 2.27

| Archivo | Qué es |
|---|---|
| `checkpoint4_*.json` | **Workflow principal** (*Modulo 4 orquestador*): es el que se evalúa |
| `workers/worker-gestion-de-turnos.json` | Worker Turnos |
| `workers/worker-identificacion-de-cliente.json` | Worker Identificación |
| `workers/soporte-errores-y-trazabilidad.json` | Soporte (*Error Workflow*) |

---

## 1. Qué contiene el workflow

El workflow contiene **dos flujos**: son **dos vías por las que un cliente puede contactarse con la empresa, por mail o por chat**.
Ambos terminan llamando a los **workers** que necesitan para desarrollar sus actividades; esos workers están en la carpeta [`workers/`](workers/).

| Worker | Qué hace | Lo usa |
|---|---|---|
| **Turnos** | Consulta disponibilidad, reserva y lista turnos en Google Calendar (y envía el mail de confirmación al reservar) | Mail y Chat |
| **Identificación** | Valida y guarda nombre, apellido, email y teléfono del cliente (NocoDB) | Chat |
| **Soporte** | Es el *Error Workflow* del orquestador: registra el error en una planilla y avisa al supervisor por mail | Ambos, ante una falla |

### Flujo 1 · Mail

1. **Gmail Trigger** recibe el mail.
2. **① ¿Respuesta automática?**: si lo es, se ignora.
3. **④ Limpiar y validar payload** (Set) y chequeo de que sea válido.
4. Aviso de inicio al equipo por Telegram.
5. **Clasificar mail** (IA: `TURNO` u `OTRO`, más fechas y datos del cliente) y **normalizar la clasificación** (JavaScript).
6. **② Buscar contacto** en el CRM (Google Contacts) y validar los datos del cliente.
7. Si es `TURNO`, se llama al **Worker Turnos**; si es `OTRO`, se arma un acuse genérico.
8. Se arma la respuesta y se **actualiza o crea el contacto**.
9. **③ Gmail: Crear borrador**: la respuesta queda en el mismo hilo para que una persona la revise (no se envía sola).
10. Resumen del caso, aviso por Telegram si es relevante y registro del mail en Google Sheets.

### Flujo 2 · Chat

1. **Chat Trigger** recibe el mensaje y se busca la **memoria de la sesión** en NocoDB. Si es un cliente nuevo, se avisa por Telegram y se lo registra.
2. Se carga la memoria y el historial, y se prepara el contexto.
3. El **Orquestador Chat** (AI Agent) hace el triaje y el código **normaliza y valida** su resultado (JavaScript).
4. El **Enrutador de Intenciones** elige el camino:
   - **Identificación**: Worker Identificación y alta o actualización del contacto en el CRM.
   - **Turnos**: Worker Turnos (ver mis turnos, ver disponibilidad, reservar).
   - **Derivación** a un supervisor.
   - **Respuesta directa**.
5. Se unifica la respuesta y, si requiere intervención humana, se alerta al equipo por Telegram.
6. Se guarda la memoria (con resumen de la conversación cuando pasa de 5 mensajes) y se responde al cliente.

- Para ver turnos, ver disponibilidad o reservar, el cliente debe estar registrado (nombre, apellido, email y teléfono).
- Para reservar, el chat primero verifica el horario y pregunta si lo reserva. Solo reserva si el cliente confirma.
- El chat es el del editor de n8n (se abre con *Open chat*).

---

## 2. Dónde está cada punto de la rúbrica

| Punto | Nodo(s) |
|---|---|
| **① IF anti auto-reply** justo después del trigger | `¿Respuesta Automática?`: descarta por asunto, por remitente (*no-reply*, *mailer-daemon*…) y por encabezados (*Auto-Submitted*, *Precedence*…) |
| **② Look up antes del Create** | Mail: `Buscar Contacto (CRM)` → `¿Contacto Existe?` → `Actualizar Contacto` / `Crear Contacto`. Chat: la misma secuencia con los nodos `(Chat)` |
| **③ Create Draft** (Human-in-the-loop) | `Gmail: Crear Borrador`. El orquestador no tiene ningún nodo *Send* |
| **④ Set de limpieza y validación** | `Limpiar y Validar Payload` (email, nombre, asunto y cuerpo sin texto citado ni HTML) + `¿Payload Válido?` |
| **Set previo al canal de mensajería** | `Armar Aviso de Inicio (Mail)`, `Payload Telegram`, `Armar Aviso de Inicio (Chat)` y `Armar Alerta (Chat)`: escapan los caracteres especiales del formato HTML de Telegram |
| **Control de errores** | Error Workflow = *Soporte - Errores y Trazabilidad* |

---

## 3. Validaciones en JavaScript y normalización de datos

En el flujo se van a ver **validaciones hechas con JavaScript** (nodos *Code* y expresiones en los nodos *Set* e *IF*). Esto se hizo así por una cuestión de **seguridad y fidelidad de los datos**: no lo estaba haciendo bien ni la IA ni los nodos esperados. La IA propone (clasifica, extrae fechas y datos) y el código decide qué se acepta.

- **`Normalizar Clasificación`** (mail): limita la intención a `TURNO` u `OTRO`, valida fechas, horas, nombre y teléfono, y corrige el día de la semana cuando el modelo convierte mal una frase como “el martes”.
- **`Normalizar Triaje (Chat)`** (chat): acepta solo acciones de una lista cerrada. Reserva únicamente si el cliente confirma y la última respuesta fue la verificación de ese horario. Productos, precios, presupuestos, cancelaciones y reprogramaciones se derivan siempre a una persona.

También se **normalizan los datos** en cada paso (email, nombre, teléfono, fechas y texto del mail) para que lo que llega a Google Contacts, Calendar, Sheets y NocoDB sea consistente.

---

## 4. Herramientas conectadas

| Herramienta | Para qué se usa | Autenticación |
|---|---|---|
| **Gmail** | Casilla de soporte: Trigger y *Create Draft*. Los workers envían mail solo para confirmar un turno y avisar al supervisor | OAuth2 |
| **Google Contacts** | CRM: buscar, actualizar y crear clientes | OAuth2 |
| **Google Calendar** | Revisión y reserva de turnos (Worker Turnos) | OAuth2 |
| **Google Sheets** | Registro de mails procesados y log de errores | OAuth2 |
| **Telegram** | Canal para contactar al equipo de soporte | Token de bot |
| **OpenAI** (`gpt-4o-mini`) | Clasificación, triaje y resumen | API key |
| **NocoDB** | Memoria persistente del chat | API token |

Se usó **Telegram en lugar de Slack** (el enunciado permite herramientas equivalentes), pensando en que a futuro se van a implementar módulos de voz.

---

## 5. Datos reemplazados por placeholders

Como el repositorio es público, **algunos IDs fueron cambiados por placeholders y por datos de ejemplo** por una cuestión de seguridad. Las credenciales no viajan en los archivos (solo el nombre de la referencia), así que no hay tokens ni claves.

| Valor en el archivo | Qué representa |
|---|---|
| `-1000000000000` | `chat_id` del chat de Telegram del equipo |
| `ID_DE_LA_PLANILLA_GOOGLE_SHEETS` | Planilla de Google Sheets |
| `ID_BASE_NOCODB` y `ID_TABLA_MEMORIA_CLIENTES` | Base y tabla de NocoDB |
| `ID_DEL_CALENDARIO@group.calendar.google.com` | Calendario de turnos |
| `supervisor@example.com` | Mail del supervisor |

---

## 6. Notas

- En el flujo mail, la IA es un **Information Extractor** (clasifica y extrae datos) y el texto del borrador se arma con plantillas en nodos *Set*. En el chat, un **AI Agent** hace el triaje y devuelve un JSON que luego valida el código.
- Como verán, el workflow fue evolucionando tanto que parece complejo, pero en realidad son **varios pasos simples**: se redactan respuestas para el usuario, se normalizan los datos, se resume y se almacena.

Cualquier duda me la hacen saber y responderé o ajustaré lo que sea necesario.
