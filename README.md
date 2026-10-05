# Checkpoint 1 – Agente autónomo básico: Asistente de Calificación de Leads

Pre-entrega 1 (Módulo 1) del curso **IA Automation Avanzado – Coderhouse**.
Autora: Camila Barrionuevo.

## Qué hace

Un agente en n8n atiende a personas que escriben a una consultora de mejora de procesos. Conversa con ellas, recolecta nombre, email, necesidad, presupuesto y plazo, las clasifica (`CALIFICADO`, `A_REVISAR`, `NO_CALIFICADO`) y registra la ficha en una hoja de Google Sheets. Cada ejecución genera además un reporte de auditoría por email.

> El contexto de negocio es ficticio y sirve para el ejercicio.

## Arquitectura

```
Chat Trigger ──► Agente Calificador de Leads ──► Armar reporte de auditoria ──► Gmail (reporte)
                   ▲   ▲   ▲                  └─(error)─► Gmail (alerta de fallo)
                   │   │   └─ Tool: registrar_lead_calificado (Google Sheets, lateral)
                   │   └──── Memoria de conversación (ventana de 10 mensajes)
                   └──────── OpenAI Chat Model
```

| Zona | Nodos | Tipo de lógica |
|---|---|---|
| Entrada | Chat Trigger | Determinista |
| Razonamiento | AI Agent (Tools Agent), Chat Model, Memoria, Tool | Probabilística (IA) |
| Observabilidad | Set + Gmail (reporte) + Gmail (alerta de fallo) | Determinista |

## Decisiones de diseño

- **Tool lateral**: la herramienta de Google Sheets está conectada al puerto `Tool` del agente, no en secuencia. El agente decide cuándo usarla según su descripción.
- **Mínimo privilegio**: la tool solo escribe (`appendOrUpdate` con el email como clave). No puede leer ni borrar leads de otros.
- **Guardrails**: límites explícitos en el System Message, regla de "No informado" contra datos inventados, frase fija de escalamiento a humano, `Max Iterations = 6`.
- **Observabilidad determinista**: el reporte no lo redacta la IA. Un nodo Set arma el estado (`Tarea completada`, `ESCALADO A HUMANO` o `ALERTA - LIMITE DE ITERACIONES ALCANZADO`) con reglas, y un Gmail lo envía. Si el agente falla, la salida de error dispara una alerta aparte.
- **Gmail en lugar de Slack**: la consigna admite Slack o Gmail; usé Gmail.

## Cómo importarlo

1. Crear una hoja de Google Sheets con una pestaña llamada `Leads` e importar `leads_template.csv` (solo trae los encabezados).
2. En n8n: *Workflows → Import from file* y elegir `checkpoint1_Camila_Barrionuevo.json`.
3. Configurar tus propias credenciales (el JSON no incluye ninguna): OpenAI, Google Sheets OAuth2 y Gmail OAuth2.
4. En el nodo `registrar_lead_calificado`, elegir tu documento y la pestaña `Leads`.
5. En los dos nodos Gmail, reemplazar `REEMPLAZAR_CON_TU_EMAIL@ejemplo.com` por tu email.
6. Ejecutar con *Execute workflow* y probar desde el chat.

## Pruebas sugeridas

| Caso | Resultado esperado |
|---|---|
| Lead con todos los datos | Fila en la hoja y estado `Tarea completada` |
| Lead sin email | El agente lo pide y no usa la tool |
| Pide una cotización | Responde "Transfiriendo tu caso a revisión humana." y estado `ESCALADO A HUMANO` |
| Tema ajeno (política, recetas) | Rechaza amablemente y no usa la tool |
| Intento de manipulación | Escala a humano |
| `Max Iterations = 1` (prueba temporal) | Estado `ALERTA - LIMITE DE ITERACIONES ALCANZADO` |

## Hoja de ruta

El proyecto crece módulo a módulo sobre este mismo workflow: multiagente (M2), memoria persistente en Airtable por `Session_ID` (M3), CRM y calendario con OAuth2 (M4), RAG (M5) y voz (M6).

## Seguridad

Este repositorio no contiene claves ni credenciales. El JSON exportado solo referencia credenciales por nombre e ID interno de n8n.
