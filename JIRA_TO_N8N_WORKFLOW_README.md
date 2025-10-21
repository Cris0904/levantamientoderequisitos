# Jira to n8n Workflow Generator

Flujo de n8n que extrae automáticamente issues de Jira y los convierte en workflows de n8n utilizando IA.

## Descripción

Este workflow automatiza la conversión de issues de Jira en especificaciones de workflows de n8n. El flujo:

1. Se ejecuta cada hora automáticamente
2. Consulta la API de Jira para obtener issues pendientes
3. Procesa cada issue extrayendo información clave
4. Utiliza OpenAI GPT-4 para generar un workflow de n8n válido basado en los requisitos del issue
5. Guarda los workflows generados en un endpoint de almacenamiento
6. Incluye manejo robusto de errores

## Estructura del Workflow

### Nodos Principales

1. **Schedule Trigger**: Ejecuta el workflow cada hora
2. **Configuration**: Define la configuración de Jira y endpoints
3. **Get Jira Issues**: Consulta la API REST de Jira
4. **Check Jira API Success**: Verifica si la llamada a Jira fue exitosa
5. **Split Issues**: Separa cada issue en items individuales
6. **Format Issue Data**: Extrae y formatea datos de cada issue
7. **Generate Workflow with AI**: Usa OpenAI GPT-4 para generar el workflow
8. **Parse Workflow JSON**: Procesa y valida el JSON generado
9. **Check Workflow Generation**: Verifica el éxito de la generación
10. **Save Generated Workflow**: Guarda el workflow generado
11. **Aggregate Results**: Agrupa todos los resultados exitosos
12. **Execution Summary**: Genera resumen de la ejecución
13. **Log Generation Errors**: Registra errores en la generación
14. **Log Jira API Error**: Registra errores de la API de Jira

## Requisitos Previos

### 1. Credenciales de Jira

Necesitas configurar autenticación HTTP Basic para Jira:

- **URL de Jira**: Tu instancia de Jira Cloud (ej: `https://tu-empresa.atlassian.net`)
- **Usuario**: Tu email de Jira
- **API Token**: Generado desde [Atlassian API tokens](https://id.atlassian.com/manage-profile/security/api-tokens)

### 2. Credenciales de OpenAI

Necesitas una API Key de OpenAI:

- **API Key**: Obtenida desde [OpenAI Platform](https://platform.openai.com/api-keys)
- Configurar como Header Auth con:
  - Header Name: `Authorization`
  - Header Value: `Bearer YOUR_API_KEY`

### 3. Endpoint de Almacenamiento

Un webhook o API endpoint donde guardar los workflows generados. Puede ser:
- Un webhook de n8n
- Una API de base de datos
- Un servicio de almacenamiento en la nube

## Instalación

### Paso 1: Importar el Workflow

1. Abre n8n
2. Haz clic en el menú (⋮) en la esquina superior derecha
3. Selecciona **Import from File**
4. Carga el archivo `jira-to-n8n-workflow.json`

### Paso 2: Configurar Credenciales

#### Jira API Credentials

1. Ve al nodo **Get Jira Issues**
2. Haz clic en **Credentials** > **Create New**
3. Selecciona **HTTP Basic Auth**
4. Configura:
   - **Credential Name**: `Jira API Credentials`
   - **User**: Tu email de Jira
   - **Password**: Tu API Token de Jira

#### OpenAI API Key

1. Ve al nodo **Generate Workflow with AI**
2. Haz clic en **Credentials** > **Create New**
3. Selecciona **Header Auth**
4. Configura:
   - **Credential Name**: `OpenAI API Key`
   - **Name**: `Authorization`
   - **Value**: `Bearer sk-...` (tu API key)

### Paso 3: Configurar Parámetros

Edita el nodo **Configuration** y actualiza:

```javascript
return {
  json: {
    jiraUrl: 'https://tu-empresa.atlassian.net',  // Tu URL de Jira
    projectKey: 'PROJ',                             // Clave de tu proyecto
    storageWebhookUrl: 'https://tu-endpoint.com'   // Endpoint de almacenamiento
  }
};
```

### Paso 4: Personalizar Consulta JQL (Opcional)

En el nodo **Get Jira Issues**, puedes modificar la consulta JQL:

```
project = {{ $json.projectKey }} AND status != Done ORDER BY created DESC
```

Ejemplos de consultas:
- `project = MYPROJ AND type = Story AND status = "To Do"`
- `project = MYPROJ AND labels = automation`
- `project = MYPROJ AND created >= -7d`

## Uso

### Ejecución Manual

1. Haz clic en **Execute Workflow** en la esquina superior derecha
2. El workflow procesará los issues inmediatamente

### Ejecución Automática

El workflow se ejecuta automáticamente cada hora según el Schedule Trigger.

Para cambiar la frecuencia:
1. Ve al nodo **Schedule Trigger**
2. Modifica el parámetro `hoursInterval` (ej: `2` para cada 2 horas)

## Estructura de Datos

### Campos Extraídos de Jira

```json
{
  "issueKey": "PROJ-123",
  "title": "Implementar autenticación OAuth",
  "description": "Descripción completa del issue",
  "issueType": "Story",
  "priority": "High",
  "status": "To Do",
  "acceptanceCriteria": "Criterios de aceptación del campo custom",
  "createdDate": "2025-01-15T10:30:00Z"
}
```

### Workflow Generado

```json
{
  "name": "Workflow para PROJ-123",
  "nodes": [...],
  "connections": {...},
  "settings": {
    "executionOrder": "v1"
  },
  "meta": {
    "sourceJiraIssue": "PROJ-123",
    "generatedAt": "2025-01-15T11:00:00Z",
    "issueTitle": "Implementar autenticación OAuth"
  }
}
```

## Personalización

### Modificar el Prompt de IA

Edita el nodo **Format Issue Data** para cambiar cómo se genera el workflow:

```javascript
const aiPrompt = `Analyze this Jira issue and create a complete n8n workflow...

// Agrega instrucciones adicionales aquí
- Use webhooks for triggers
- Include error handling
- Add data validation
`;
```

### Cambiar Modelo de IA

En el nodo **Generate Workflow with AI**, cambia el modelo:

```json
{
  "name": "model",
  "value": "gpt-4-turbo"  // o "gpt-3.5-turbo" para mayor velocidad
}
```

### Filtrar Campos Custom de Jira

Modifica el parámetro `fields` en **Get Jira Issues**:

```
summary,description,issuetype,priority,status,customfield_10000,customfield_10001
```

Para encontrar IDs de campos custom:
1. Ve a Jira
2. Inspecciona la página del issue
3. O usa: `https://tu-empresa.atlassian.net/rest/api/3/field`

## Monitoreo y Logs

### Ver Ejecuciones

1. Ve a **Executions** en el panel lateral de n8n
2. Haz clic en una ejecución para ver detalles

### Logs de Errores

Los errores se registran en:
- **Log Jira API Error**: Errores al consultar Jira
- **Log Generation Errors**: Errores al generar workflows

### Validación de Workflows

El nodo **Parse Workflow JSON** valida que el JSON generado sea válido y tenga la estructura correcta de n8n.

## Troubleshooting

### Error: "Failed to fetch Jira issues"

**Causa**: Credenciales incorrectas o URL inválida

**Solución**:
1. Verifica las credenciales de Jira
2. Confirma que la URL es correcta
3. Verifica que el API Token sea válido

### Error: OpenAI rate limit

**Causa**: Límite de solicitudes de OpenAI excedido

**Solución**:
1. Reduce la frecuencia del Schedule Trigger
2. Disminuye `maxResults` en la consulta de Jira
3. Usa `gpt-3.5-turbo` que tiene límites más altos

### Workflows generados vacíos

**Causa**: El modelo de IA no generó JSON válido

**Solución**:
1. Revisa los logs del nodo **Parse Workflow JSON**
2. Mejora el prompt en **Format Issue Data**
3. Aumenta `max_tokens` en la llamada a OpenAI

## Extensiones Posibles

### 1. Guardar en Base de Datos

Reemplaza **Save Generated Workflow** con un nodo de base de datos (PostgreSQL, MongoDB, etc.)

### 2. Notificaciones

Agrega un nodo de Slack o Gmail después de **Execution Summary** para recibir notificaciones

### 3. Validación Avanzada

Agrega un nodo Code después de **Parse Workflow JSON** para validar:
- Que los nodos tengan parámetros completos
- Que las conexiones sean válidas
- Que no falten credenciales

### 4. Importación Automática

Usa la API de n8n para importar automáticamente los workflows generados:

```javascript
// Nuevo nodo HTTP Request
POST https://tu-n8n.com/api/v1/workflows
Authorization: Bearer YOUR_N8N_API_KEY
Body: $json.workflow
```

## Recursos

- [Documentación de n8n](https://docs.n8n.io/)
- [Jira REST API v3](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)
- [JQL Query Syntax](https://www.atlassian.com/software/jira/guides/jql)

## Licencia

Este workflow es de uso libre. Puedes modificarlo según tus necesidades.

## Soporte

Para problemas o sugerencias:
1. Revisa los logs de ejecución en n8n
2. Consulta la documentación oficial de n8n
3. Verifica las credenciales y configuraciones

---

**Creado con n8n Workflow Assistant MCP**
