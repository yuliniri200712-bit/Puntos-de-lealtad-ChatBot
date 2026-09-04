# Puntos-de-lealtad-ChatBot

# DeliveryBot - Cafetería · Función de Puntos de Lealtad

Documentación de la funcionalidad **Programa de Puntos de Lealtad** implementada en el workflow de n8n `DeliveryBot - Cafeteria (Puntos de Lealtad)`.

---

## 1. Descripción general

DeliveryBot es un asistente conversacional para una cafetería que opera dentro de **Telegram**. Además de gestionar pedidos, ahora incorpora un **programa de puntos de lealtad**: cada pedido confirmado otorga puntos al cliente, que quedan acumulados y asociados a su cuenta de Telegram para futuros canjes de premios.

El bot está construido sobre un **AI Agent** (modelo `gpt-5-mini`) que interpreta los mensajes del usuario y decide qué herramientas ejecutar. Toda la información persiste en un **Google Sheets** (`DeliveryBot_DB`) que actúa como base de datos.

---

## 2. Arquitectura del workflow

Telegram Trigger ──▶ DeliveryBot Agent ──▶ Responder al Cliente (Telegram)
│
┌───────────────────┼──────────────────────────────┐
│ │ │
OpenAI Chat Model Session Memory Herramientas (Google Sheets)
(gpt-5-mini) (buffer, 10 msgs) get_menu · create_order · get_order_status
get_user_points · update_user_points


| Componente | Tipo | Función |
|------------|------|---------|
| **Telegram Trigger** | Trigger | Recibe los mensajes entrantes del usuario. |
| **DeliveryBot Agent** | AI Agent | Cerebro conversacional; orquesta las herramientas. |
| **OpenAI Chat Model** | Modelo LLM | `gpt-5-mini`, razonamiento del agente. |
| **Session Memory** | Memoria | Buffer de ventana (10 mensajes), sesión por `chat.id`. |
| **Responder al Cliente** | Telegram | Envía la respuesta final al chat. |

---

## 3. Base de datos (Google Sheets: `DeliveryBot_DB`)

La función de lealtad utiliza principalmente la hoja **USUARIOS**.

### Hoja `USUARIOS`
| Columna | Tipo | Descripción |
|---------|------|-------------|
| `telegram_id` | texto | Identificador único del usuario en Telegram (clave de coincidencia). |
| `puntos_lealtad` | número | Balance actual de puntos acumulados del usuario. |

### Hoja `PEDIDOS` (relacionada)
| Columna | Descripción |
|---------|-------------|
| `id_pedido` | ID único del pedido. |
| `id_usuario` | `telegram_id` del cliente. |
| `nombre` | Nombre del cliente. |
| `detalles_pedido` | Productos y cantidades. |
| `total_pago` | Total del pedido (usado para calcular los puntos). |
| `estado` | Estado del pedido (inicial: `Recibido`). |
| `fecha` / `hora` | Fecha y hora del registro. |

### Hoja `MENU`
Bebidas, comidas y snacks con precios y stock (usada por `get_menu`).

---

## 4. Herramientas del sistema de puntos

| Herramienta | Operación en Sheets | Hoja | Rol en la lógica de lealtad |
|-------------|---------------------|------|------------------------------|
| **get_user_points** | `read` (primer match por `telegram_id`) | USUARIOS | Obtiene el nombre y el balance actual de `puntos_lealtad`. |
| **update_user_points** | `update` (match por `telegram_id`) | USUARIOS | Guarda el **nuevo total** de `puntos_lealtad`. |
| **create_order** | `append` | PEDIDOS | Registra el pedido cuyo `total_pago` alimenta el cálculo de puntos. |

---

## 5. Lógica de acumulación de puntos

Se ejecuta **por cada pedido confirmado**:

1. **Cálculo de puntos ganados**
   - Regla: **1 punto por cada $5.00 gastados**.
   - Fórmula: `puntos_ganados = floor(total_pago / 5)`
   - Ejemplo: un pedido de `$15.00` → `floor(15 / 5)` = **3 puntos**.

2. **Lectura del balance actual**
   - Con `get_user_points` se obtiene el valor de `puntos_lealtad` del usuario según su `telegram_id`.

3. **Actualización del balance**
   - Nuevo total = `balance_actual + puntos_ganados`.
   - Con `update_user_points` se guarda el **nuevo total** en la hoja USUARIOS (coincidencia por `telegram_id`).

> Importante: `update_user_points` siempre almacena el **balance total resultante**, no solo los puntos ganados en el pedido.

---

## 6. Flujo "Realizar pedido" (con puntos)

1. El agente muestra el menú con `get_menu` y ayuda a elegir productos y cantidades.
2. Confirma el pedido con el total antes de registrarlo
   (ej.: *"Llevas 1 Café + 1 Empanada. Total $15.00. ¿Confirmas?"*).
3. Al confirmar:
   - `create_order` registra el pedido con `id_usuario = telegram_id`, fecha y hora actuales, y estado `Recibido`.
   - Se calculan los puntos con `floor(total / 5)`.
   - `get_user_points` → obtiene el balance actual.
   - `update_user_points` → guarda el nuevo total (balance + puntos ganados).

---

## 7. Opción "4. Ver mis Puntos"

Cuando el usuario elige consultar sus puntos:

1. El agente usa `get_user_points` con el `telegram_id` del usuario.
2. Responde con el formato exacto:

Hola [Nombre], actualmente tienes puntos: [Puntos] puntos acumulados. Sigue comprando para canjear premios!


donde `[Nombre]` es el nombre del usuario y `[Puntos]` su balance de `puntos_lealtad`.

---

## 8. Menú principal del bot

Al saludar o pedir ayuda, el bot ofrece:

1. Ver el menú
2. Realizar un pedido
3. Consultar el estado de mi pedido
4. **Ver mis Puntos**

---

## 9. Datos de contexto disponibles para el agente

En cada interacción el agente conoce:

- **telegram_id** del usuario: `message.from.id`
- **Nombre** del usuario: `message.from.first_name`
- **Fecha** actual: formato `yyyy-MM-dd`
- **Hora** actual: formato `HH:mm`

---

## 10. Requisitos / Credenciales

- **Telegram Bot** (Trigger y envío de mensajes).
- **Google Sheets** (acceso al documento `DeliveryBot_DB`).
- **OpenAI** (modelo `gpt-5-mini`).

---

## 11. Ejemplo de cálculo

| Pedido | `total_pago` | Puntos ganados `floor(total/5)` | Balance previo | Nuevo balance |
|--------|--------------|-------------------------------|----------------|----------------|
| Café + Empanada | $15.00 | 3 | 10 | 13 |
| Jugo | $4.00 | 0 | 13 | 13 |
| Combo | $22.00 | 4 | 13 | 17 |
