# Guía de Inicio: Amazon Bedrock AgentCore

Esta guía te ayudará a construir y desplegar un agente de IA listo para producción en minutos utilizando Amazon Bedrock AgentCore.

## Tabla de Contenidos

1. [Requisitos Previos](#requisitos-previos)
2. [Instalación del Toolkit](#instalación-del-toolkit)
3. [Crear el Agente](#crear-el-agente)
4. [Configurar y Desplegar](#configurar-y-desplegar)
5. [Monitorear el Despliegue](#monitorear-el-despliegue)
6. [Probar Memoria y Code Interpreter](#probar-memoria-y-code-interpreter)
7. [Ver Trazas y Logs](#ver-trazas-y-logs)
8. [Limpieza de Recursos](#limpieza-de-recursos)
9. [Solución de Problemas](#solución-de-problemas)

---

## Requisitos Previos

Antes de comenzar, asegúrate de tener:

### 1. Permisos de AWS
- Usuarios root de AWS o usuarios con roles privilegiados (como `AdministratorAccess`) pueden omitir este paso
- Otros usuarios necesitan adjuntar la política del starter toolkit y la política administrada `AmazonBedrockAgentCoreFullAccess`

### 2. AWS CLI
- Versión 2.0 o posterior
- Configurar usando: `aws configure`

### 3. Acceso al Modelo Amazon Bedrock
- Habilitar acceso al modelo **Claude 3.7 Sonnet**
- Ir a AWS Management Console → Amazon Bedrock → Model access
- Habilitar **Claude 3.7 Sonnet** en tu región de AWS

### 4. Python
- Python 3.10 o más reciente

### 5. Consistencia de Región AWS
⚠️ **IMPORTANTE**: Asegúrate de usar la **misma región de AWS** para:
- La región predeterminada seleccionada en `aws configure`
- La región donde habilitaste el acceso al modelo de Amazon Bedrock

---

## Instalación del Toolkit

### Paso 1: Crear Entorno Virtual

```bash
# Crear entorno virtual
python -m venv .venv

# Activar entorno virtual
# En macOS/Linux:
source .venv/bin/activate

# En Windows:
# .venv\Scripts\activate
```

### Paso 2: Instalar Paquetes Requeridos

```bash
# Instalar paquetes (versión 0.1.21 o posterior)
pip install "bedrock-agentcore-starter-toolkit>=0.1.21" strands-agents boto3
```

---

## Crear el Agente

### Paso 1: Crear el Archivo del Agente

Crea un archivo llamado `agentcore_starter_strands.py` con el código del agente.

**Características del agente:**
- Integración con AgentCore Memory (memoria a corto y largo plazo)
- Herramienta de cálculo con Code Interpreter
- Gestión de sesiones
- Sistema de prompts personalizable

### Paso 2: Crear Archivo de Requisitos

Crea un archivo `requirements.txt`:

```txt
strands-agents
bedrock-agentcore
```

---

## Configurar y Desplegar

### Paso 1: Configurar el Agente

Ejecuta el comando de configuración:

```bash
agentcore configure -e agentcore_starter_strands.py
```

**Responde a los prompts interactivos:**

1. **Execution Role**: Presiona Enter para auto-crear un nuevo rol con todos los permisos requeridos
2. **ECR Repository**: Presiona Enter para auto-crear o proporciona un URI existente
3. **Requirements File**: Confirma el archivo `requirements.txt` detectado
4. **OAuth Configuration**: Escribe `no` para este tutorial
5. **Request Header Allowlist**: Escribe `no` para este tutorial
6. **Memory Configuration**:
   - Si se encuentran memorias existentes: Elige de la lista o presiona Enter para crear nueva
   - Si creas nueva: ¿Habilitar extracción de memoria a largo plazo? Escribe `yes`
   - Nota: La memoria a corto plazo está siempre habilitada por defecto

### Paso 2: Desplegar en AgentCore

Lanza tu agente al entorno de ejecución de AgentCore:

```bash
agentcore launch
```

**Este comando realiza:**
1. Aprovisionamiento de recursos de memoria (estrategias STM + LTM)
2. Construcción del contenedor Docker con dependencias
3. Push al repositorio ECR
4. Despliegue de AgentCore Runtime con trazabilidad X-Ray habilitada
5. Configuración de CloudWatch Transaction Search (automática)
6. Activación del endpoint con recolección de trazas

**Salida esperada:**
```
✅ Memory created: bedrock_agentcore_memory_ci_agent_memory-abc123
Observability is enabled, configuring Transaction Search...
✅ Transaction Search configured: resource_policy, trace_destination, indexing_rule
🔍 GenAI Observability Dashboard:
   https://console.aws.amazon.com/cloudwatch/home?region=us-west-2#gen-ai-observability/agent-core
✅ Container deployed to Bedrock AgentCore
Agent ARN: arn:aws:bedrock-agentcore:us-west-2:123456789:runtime/starter_agent-xyz
```

### Paso 3: Verificar Configuración (si hay errores)

Si el despliegue encuentra errores:

```bash
# Revisar configuración desplegada
cat .bedrock_agentcore.yaml

# Verificar estado de aprovisionamiento de recursos
agentcore status
```

---

## Monitorear el Despliegue

### Verificar Estado del Agente

```bash
agentcore status
```

**Información mostrada:**
- Memory ID
- Memory Status (CREATING → ACTIVE)
- Memory Type (STM+LTM con estrategias)
- Agent ARN
- Observability Dashboard URL
- Runtime logs path

### Esperar hasta que la Memoria esté Activa

⏱️ El aprovisionamiento de memoria toma **2-3 minutos**

```bash
# Verificar cada 30 segundos hasta que el estado sea ACTIVE
agentcore status
```

---

## Probar Memoria y Code Interpreter

### Probar Memoria a Corto Plazo (STM)

La memoria a corto plazo mantiene contexto dentro de una sesión:

```bash
# Primera invocación - almacenar información
agentcore invoke '{"prompt": "Mi nombre es Juan"}'

# Segunda invocación en la misma sesión - recuperar información
agentcore invoke '{"prompt": "¿Cuál es mi nombre?"}'
```

**Respuesta esperada:** "Tu nombre es Juan."

### Probar Memoria a Largo Plazo (LTM)

La memoria a largo plazo persiste hechos entre sesiones diferentes.

⏱️ **Importante:** AgentCore ejecuta extracciones en segundo plano. Espera 10-30 segundos después de almacenar hechos.

```bash
# Sesión 1: Almacenar hechos
agentcore invoke '{"prompt": "Mi email es usuario@ejemplo.com y soy un usuario de AgentCore"}'

# Esperar a que la extracción finalice
sleep 20

# Sesión 2: Nueva sesión diferente recupera los hechos extraídos
SESSION_ID=$(python -c "import uuid; print(uuid.uuid4())")
agentcore invoke '{"prompt": "¿Qué sabes sobre mí?"}' --session-id $SESSION_ID
```

**Respuesta esperada:**
- "Tu dirección de email es usuario@ejemplo.com."
- "Pareces ser un usuario de AgentCore."

### Probar Code Interpreter

El Code Interpreter permite ejecutar código Python de forma segura:

```bash
# Almacenar datos
agentcore invoke '{"prompt": "Mi dataset tiene valores: 23, 45, 67, 89, 12, 34, 56."}'

# Crear visualización
agentcore invoke '{"prompt": "Crea una visualización de gráfico de barras basada en texto que muestre la distribución de valores en mi dataset con etiquetas apropiadas"}'
```

**Resultado esperado:** El agente genera código matplotlib para crear un gráfico de barras.

---

## Ver Trazas y Logs

### Acceder al Dashboard de Amazon CloudWatch

El dashboard GenAI Observability proporciona:
- Trazas end-to-end de solicitudes
- Seguimiento de ejecución del agente
- Operaciones de recuperación de memoria
- Ejecuciones de code interpreter
- Pasos de razonamiento del agente
- Desglose de latencia por componente

```bash
# Obtener la URL del dashboard desde status
agentcore status

# O ir directamente a:
# https://console.aws.amazon.com/cloudwatch/home?region=YOUR-REGION#gen-ai-observability/agent-core
# Nota: Reemplaza YOUR-REGION con tu región
```

### Ver Logs de AgentCore Runtime

Accede a logs detallados para debugging y monitoreo:

```bash
# Las rutas correctas de logs se muestran en la salida de invoke o status
agentcore status

# Verás rutas de logs como:
# aws logs tail /aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT --log-stream-name-prefix "YYYY/MM/DD/[runtime-logs]" --follow

# Copia este comando de la salida para ver logs
# Ejemplo:
aws logs tail /aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT \
  --log-stream-name-prefix "YYYY/MM/DD/[runtime-logs]" \
  --follow

# Para logs recientes, usa la opción --since:
aws logs tail /aws/bedrock-agentcore/runtimes/AGENT_ID-DEFAULT \
  --log-stream-name-prefix "YYYY/MM/DD/[runtime-logs]" \
  --since 1h
```

---

## Limpieza de Recursos

Para eliminar todos los recursos creados durante este tutorial:

```bash
agentcore destroy
```

**Este comando elimina:**
- Endpoint y agente de AgentCore Runtime
- Recursos de AgentCore Memory (memoria a corto y largo plazo)
- Repositorio e imágenes de Amazon ECR
- Roles IAM (si fueron auto-creados)
- Grupos de logs de CloudWatch (opcional)

---

## Solución de Problemas

### Problema: Opción de memoria no aparece durante `agentcore configure`

**Causa:** Versión desactualizada del toolkit (necesitas >= 0.1.21)

**Solución:**

```bash
# Paso 1: Verificar estado actual
which python   # Debería mostrar .venv/bin/python
which agentcore  # Actualmente mostrando ruta global

# Paso 2: Desactivar y reactivar venv para reiniciar PATH
deactivate
source .venv/bin/activate

# Paso 3: Verificar si se solucionó
which agentcore
# Si AHORA muestra .venv/bin/agentcore -> RESUELTO, salta al Paso 7
# Si TODAVÍA muestra ruta global -> continúa al Paso 4

# Paso 4: Forzar que el venv local tome precedencia en PATH
export PATH="$(pwd)/.venv/bin:$PATH"

# Paso 5: Verificar de nuevo
which agentcore
# Si AHORA muestra .venv/bin/agentcore -> RESUELTO, salta al Paso 7
# Si TODAVÍA muestra ruta global -> continúa al Paso 6

# Paso 6: Reinstalar en venv local con precedencia forzada
pip install --force-reinstall --no-cache-dir "bedrock-agentcore-starter-toolkit>=0.1.21"

# Paso 7: Verificación final
which agentcore  # Debe mostrar: /path/to/your-project/.venv/bin/agentcore
pip show bedrock-agentcore-starter-toolkit  # Verificar versión >= 0.1.21
agentcore --version  # Verificar que funciona

# Paso 8: Intentar configurar de nuevo
agentcore configure -e agentcore_starter_strands.py
```

**Verificaciones adicionales:**
- Asegúrate de ejecutar `agentcore configure` desde dentro del entorno virtual activado
- Si usas un IDE (VSCode, PyCharm), reinicia el IDE después de reinstalar
- Verifica que no hay conflictos con instalaciones globales: `pip list | grep bedrock-agentcore`

### Problema: Cambiar configuración de región AWS

```bash
# 1. Limpiar recursos en la región incorrecta
agentcore destroy

# 2. Verificar que AWS CLI esté configurado para la región correcta
aws configure get region

# O reconfigurar para la región correcta:
aws configure set region tu-region-deseada

# 3. Asegurarse de que el acceso al modelo de Amazon Bedrock esté habilitado
# Ir a: AWS Console → Amazon Bedrock → Model access

# 4. Copiar código del agente y requirements.txt
# Volver al paso "Configurar y Desplegar"
```

### Problema: Error "Memory status is not active"

```bash
# Ejecutar status para verificar el estado de memoria
agentcore status

# Si el estado muestra 'provisioning', espera 2-3 minutos
# Reintentar después de que el estado muestre 'Memory Type: STM+LTM (3 strategies)'
```

### Problema: Memoria cross-session no funciona

**Soluciones:**
- Verificar que la memoria a largo plazo esté activa (no "provisioning")
- Esperar 15-30 segundos después de almacenar hechos para la extracción
- Verificar logs de extracción para confirmar completitud

### Problema: No aparecen trazas

**Soluciones:**
- Verificar que observability fue habilitada durante `agentcore configure`
- Verificar que los permisos IAM incluyen acceso a CloudWatch y X-Ray
- Esperar 30-60 segundos para que las trazas aparezcan en CloudWatch
- Las trazas son visibles en: AWS Console → CloudWatch → Service Map o X-Ray → Traces

### Problema: Logs de memoria faltantes

```bash
# Verificar que el grupo de logs existe:
# /aws/vendedlogs/bedrock-agentcore/memory/APPLICATION_LOGS/memory-id

# Verificar que el rol IAM tiene permisos de CloudWatch Logs
```

---

## Resumen

Has desplegado un agente de producción con:

✅ **AgentCore Runtime** para orquestación de contenedores administrada

✅ **AgentCore Memory** con:
- Memoria a corto plazo para contexto inmediato
- Memoria a largo plazo para persistencia entre sesiones

✅ **AgentCore Code Interpreter** para ejecución segura de Python con capacidades de visualización de datos

✅ **AWS X-Ray Tracing** configurado automáticamente para trazabilidad distribuida

✅ **Integración con CloudWatch** para logs y métricas con Transaction Search habilitado

Todos los servicios están automáticamente instrumentados con trazabilidad X-Ray, proporcionando visibilidad completa del comportamiento del agente, operaciones de memoria y ejecuciones de herramientas a través del dashboard de CloudWatch.

---

## Recursos Adicionales

- [Documentación Oficial de AWS Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [Strands Agents SDK Documentation](https://github.com/anthropics/anthropic-sdk-python)
- [AWS CLI User Guide](https://docs.aws.amazon.com/cli/latest/userguide/)

---

**Fecha de creación:** Octubre 2025  
**Versión del toolkit:** >= 0.1.21

