# Runbook — Configuración de LM Studio (Mac mini M4, 16 GB)

## 1. Propósito y alcance

Este runbook deja operable el servidor de inferencia local que consume el agente `pentest-agent` durante la PoC. Cubre instalación, selección de modelo, configuración de contexto, exposición del servidor compatible con OpenAI y verificación de extremo a extremo desde el contenedor Docker.

**Audiencia:** operador del lab.
**Plataforma objetivo:** Mac mini M4 con 16 GB de memoria unificada y 512 GB de SSD, macOS Sequoia 15.x o posterior.
**Resultado esperado:** `curl http://host.docker.internal:1234/v1/models` responde con un modelo cargado, tanto desde el host como desde dentro del contenedor `pentest-agent`.

## 2. Requisitos de hardware y sistema

| Recurso | Mínimo | Recomendado para la PoC | Comentario |
|---|---|---|---|
| Chip | Apple M4 | Apple M4 (10-core CPU, 10-core GPU) | Soporte Metal requerido |
| Memoria unificada | 16 GB | 16 GB | LM Studio consume RAM del pool unificado |
| Almacenamiento | 20 GB libres | 100 GB libres | Modelos cuantizados + cache |
| macOS | 15.0 | 15.x (Sequoia) | Requiere Metal 3 |
| Conexión de red | Banda ancha | Banda ancha | Solo para descarga inicial de modelos |

**Regla de oro de memoria:** del total de 16 GB, dejar 4–5 GB para macOS y otras apps. Quedan ~11–12 GB para modelo + KV cache. No cargar un modelo cuyo tamaño supere 10 GB con un contexto grande.

## 3. Instalación de LM Studio

1. Descargar LM Studio desde `https://lmstudio.ai/download` (canal estable, no las nightly builds).
2. Mover `LM Studio.app` a `/Applications`.
3. Abrir LM Studio una vez para que macOS lo catalogue (Gatekeeper, permisos de acceso a `~/Documents`, etc.).
4. Confirmar versión desde la barra superior: `LM Studio → About LM Studio`. Anotar el número de build.
5. Actualizar a la última versión del catálogo interno: `Developer → Check for Updates`.

LM Studio persiste sus modelos y configuración bajo `~/.cache/lmstudio/`. Respaldar esa carpeta antes de cambios mayores.

## 4. Selección del modelo

### 4.1 Modelo primario recomendado: Qwen3 8B Instruct (Q4_K_M)

Justificación:

- La propuesta del agente ya nombra Qwen3, lo que mantiene coherencia con la planificación.
- Qwen3 8B Q4_K_M pesa ~5 GB y deja ~6–7 GB para contexto y KV cache en una máquina de 16 GB.
- Rinde bien en instrucciones estructuradas, JSON y razonamiento paso a paso, que es exactamente lo que necesita el planificador del agente.
- Cuantización Q4_K_M es el sweet spot de calidad/tamaño para 16 GB de RAM.

Descarga desde la pestaña **Discover** (icono de búsqueda) de LM Studio:

- Modelo: `Qwen3 8B` (verificar que sea la variante **Instruct**, no base).
- Cuantización: `Q4_K_M` (alternativa aceptable: `Q5_K_M` si la memoria lo permite).
- Tamaño esperado: ~4.7–5.0 GB en disco.

### 4.2 Modelos alternativos

Si la calidad de planificación de Qwen3 8B no alcanza, evaluar en este orden. Cada alternativa cambia el balance memoria ↔ capacidad.

| Modelo | Cuantización | Tamaño aprox. | Cuándo usarlo |
|---|---|---|---|
| Qwen3 14B | Q3_K_M | ~7.5 GB | Más capacidad de razonamiento, contexto más chico |
| Qwen3 14B | IQ3_XXS | ~6.0 GB | Si se necesita el headroom de contexto por encima de calidad |
| Llama 3.1 8B Instruct | Q4_K_M | ~4.7 GB | Alternativa madura, mejor adherence a function calling |
| Mistral 7B Instruct v0.3 | Q4_K_M | ~4.1 GB | Más rápido, menor calidad en instrucciones largas |
| Phi-3.5 mini (3.8B) | Q4_K_M | ~2.5 GB | Si se quiere respuesta muy rápida, baja calidad de razonamiento |
| gpt-oss-20b | Q4_K_M | ~11 GB | Solo si se reduce contexto a 4096; queda poco headroom |

**No recomendado en 16 GB:** cualquier modelo > 13 GB o cualquier cuantización Q6/Q8 sobre modelos ≥ 14B. El sistema hará swapping y la latencia se vuelve inutilizable.

### 4.3 Procedimiento de descarga

1. En LM Studio, **Search (Discover)** → buscar el nombre exacto.
2. Filtrar por **Instruct** cuando aplique.
3. Seleccionar la cuantización listada arriba.
4. Click en **Download**. Verificar tamaño en la pestaña **My Models**.
5. Una vez descargado, el modelo aparece bajo **My Models** listo para cargarse.

## 5. Carga del modelo y configuración de contexto

### 5.1 Cargar el modelo

1. En **My Models**, seleccionar `Qwen3 8B Instruct (Q4_K_M)`.
2. Configurar los parámetros antes de cargar:

   - **Context Length:** `8192` tokens.
     - Justificación: 8192 es suficiente para el prompt estructurado del agente (instrucciones + datos fetched + tool output resumido). Subir a 16384 duplica el uso de KV cache y reduce el throughput.
     - Si el agente reporta errores de contexto lleno durante las pruebas, subir a `12288` solo si la memoria lo permite.
   - **GPU Offload:** `Maximum` (offload de todas las capas a la GPU Metal del M4).
   - **Flash Attention:** `Enabled`.
   - **KV Cache Quantization:** `Q8_0` (reduce memoria del cache con pérdida de calidad casi nula; alternativa `Q4_0` si se necesita más headroom).
   - **Batch Size:** `512` (default razonable).
   - **Threads:** `Auto` (LM Studio usa todos los performance cores).

3. Click en **Load Model**. Verificar en la barra superior que aparece el nombre del modelo y el uso de memoria (esperado: ~5.5–6.5 GB total cargado con KV Q8_0).

### 5.2 Probar el modelo cargado

Abrir el panel **Chat (AI Chat)** y enviar dos pruebas rápidas:

```
Usuario: Responde únicamente con un objeto JSON válido: {"ok": true, "model": "Qwen3"}
```

Esperado: el modelo devuelve el JSON exacto. Esto valida que la salida estructurada funciona.

```
Usuario: Genera una lista de tres encabezados HTTP de seguridad ausentes en una respuesta HTTP típica.
```

Esperado: tres encabezados concretos (por ejemplo `Strict-Transport-Security`, `X-Content-Type-Options`, `Content-Security-Policy`). Esto valida razonamiento sobre seguridad web, relevante para el caso de uso.

## 6. Configuración del servidor OpenAI-compatible

El agente consume el modelo vía HTTP, no desde la UI de chat. Hay que habilitar el servidor local integrado en LM Studio.

1. En el panel izquierdo, abrir **Developer** (icono `<>`).
2. En la sección **Server Settings**:
   - **Port:** `1234` (default, mantener).
   - **Host:** `0.0.0.0` (escuchar en todas las interfaces). Esto es necesario para que el contenedor Docker alcance el servidor vía `host.docker.internal`. Si se deja en `127.0.0.1`, solo el host podrá conectarse.
   - **CORS:** `Enabled` (necesario si en el futuro se conecta una UI web).
   - **Verbose Logging:** `Enabled` durante la PoC para depurar.
3. En **Model to Serve**, seleccionar el modelo recién cargado.
4. Click en **Start Server**. Confirmar en el log que aparece `Server listening on http://0.0.0.0:1234`.
5. Mantener LM Studio abierto y con el servidor corriendo durante toda la sesión del agente.

### 6.2 (Opcional) Habilitar autenticación con API Token

> **Requiere LM Studio 0.4.0 o superior.** Versiones anteriores no tienen esta opción.

Por defecto LM Studio no requiere autenticación: cualquiera que alcance `0.0.0.0:1234` puede usar el server. En una LAN o host compartido, eso es un problema. LM Studio puede generar y validar tokens propios.

**Habilitar la validación:**

1. En la pestaña **Developer**, sección **Server Settings**, tildar el switch **"Require Authentication"** (también llamado "Enable API Token authentication" según la versión).
2. Click en **Manage Tokens**. Se abre un modal con los tokens existentes (lista vacía la primera vez).
3. Click en **Create Token**. Darle un nombre (ej. `pentest-agent-local`) y seleccionar permisos.
4. **Copiar el token inmediatamente.** LM Studio no lo vuelve a mostrar.
5. Restart del server (Stop → Start) para que tome el cambio.

**Uso desde el agente:** el token va en el header `Authorization: Bearer <token>`. En el `config/config.yaml`:

```yaml
provider:
  name: lmstudio
  base_url: http://host.docker.internal:1234/v1
  api_key: env:LM_API_TOKEN     # recomendado: variable de entorno
  model: qwen/qwen3-8b
  timeout_seconds: 120
```

Con `env:LM_API_TOKEN`, el loader lee el valor de la variable `LM_API_TOKEN` del shell al iniciar el contenedor. Definila antes de correr:

```bash
# En tu shell, antes del docker run / docker compose up:
export LM_API_TOKEN="el-token-que-copiaste-de-LM-Studio"
```

**Sin variable de entorno (NO recomendado, queda en el repo):**

```yaml
api_key: "el-token-que-copiaste-de-LM-Studio"
```

**Cuándo NO habilitar la autenticación:** si el server LM Studio escucha solo en `localhost` (Host `127.0.0.1`) y el contenedor del agente lo alcanza vía `host.docker.internal:1234` en el mismo equipo, el riesgo de exposición es bajo. Habilitar auth es una decisión de threat model, no de defaults.

## 7. Verificación de la configuración

### 7.1 Desde el host

```bash
# Verificar que el servidor responde y lista modelos.
curl -s http://localhost:1234/v1/models | jq

# Verificar que el endpoint de chat completions responde.
curl -s http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-8b",
    "messages": [{"role": "user", "content": "Responde solo: OK"}],
    "max_tokens": 8
  }' | jq
```

Salida esperada del segundo comando: un objeto JSON con `choices[0].message.content` igual a `"OK"` (o muy cercano).

### 7.2 Desde el contenedor `pentest-agent`

Una vez que la imagen del agente esté construida (tarea 1.3 de `tasks.md`):

```bash
docker compose run --rm pentest-agent \
  curl -s http://host.docker.internal:1234/v1/models | jq
```

Si `host.docker.internal` no resuelve (raro en macOS pero posible en versiones viejas de Docker Desktop), usar la IP del gateway:

```bash
# Obtener la IP del host desde el punto de vista del contenedor.
docker compose run --rm pentest-agent ip route | awk '/default/ {print $3}'

# Probar contra esa IP.
docker compose run --rm pentest-agent \
  curl -s http://<IP-DEL-HOST>:1234/v1/models | jq
```

Documentar la IP de fallback en la configuración del agente para no depender solo del hostname.

### 7.3 Verificación de carga sostenida

Durante una corrida real del agente, monitorear:

- **Activity Monitor → Memory:** LM Studio debería mostrar un uso estable entre 5.5 y 8 GB. Si supera 11 GB, hay swapping.
- **Activity Monitor → Energy → GPU:** confirmar uso no cero. Si está en 0%, las capas no se están descargando al GPU.
- **LM Studio → Developer → Server log:** buscar `tokens per second`. Esperado: 15–40 t/s para Qwen3 8B Q4_K_M en M4 con contexto 8192. Por debajo de 10 t/s, recargar el modelo con KV cache quantization Q4_0 o bajar contexto a 4096.

## 8. Configuración del agente para apuntar a LM Studio

El agente lee la URL del proveedor desde configuración. En el archivo `pentesting/config/config.yaml`:

```yaml
provider:
  name: lmstudio
  base_url: http://host.docker.internal:1234/v1
  # Si auth está habilitada en LM Studio (ver §6.2), usar:
  #   api_key: env:LM_API_TOKEN
  # Si auth está deshabilitada, cualquier string sirve:
  api_key: not-required
  model: qwen3-8b
  timeout_seconds: 120
```

Notas:

- `api_key`: por defecto LM Studio (>= 0.4.0) no valida el header. Si habilitaste autenticación en el server (Developer → Server Settings → Require Authentication), el valor tiene que coincidir con un token generado desde Manage Tokens — ver §6.2. Si el server está expuesto en LAN o compartido, habilitá auth.
- El cliente OpenAI-compatible del agente siempre envía `Authorization: Bearer <api_key>`. Si LM Studio tiene auth deshabilitada, ignora el valor; si la tiene habilitada, valida contra el token.
- Si se usa fallback a IP del host, reemplazar `host.docker.internal` por la IP documentada en 7.2.
- `timeout_seconds` debe ser generoso (≥ 120) porque el primer token en modelos locales puede tardar varios segundos.

## 9. Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Connection refused` desde el contenedor | Servidor no escuchando en `0.0.0.0` | Sección 6, cambiar Host a `0.0.0.0` y reiniciar el server |
| `host.docker.internal` no resuelve | Docker Desktop versión vieja o Linux | Sección 7.2, usar IP del gateway |
| `model_not_found` | El `model` enviado por el agente no coincide con el cargado en LM Studio | Listar con `curl http://localhost:1234/v1/models` y copiar el `id` exacto a la config |
| Latencia > 30 s por respuesta | KV cache muy grande, GPU no usada | Bajar contexto a 4096, activar Flash Attention, confirmar GPU offload = Maximum |
| LM Studio crashea al cargar | Modelo > 11 GB + macOS压力大 | Volver a la cuantización Q4_K_M o Q3_K_M, reiniciar Mac para liberar memoria |
| Outputs del modelo no son JSON válido | Modelo pequeño sin suficiente instruction tuning | Probar Llama 3.1 8B Instruct, o subir temperatura a 0 para reducir varianza |
| El agente se queda colgado en la primera llamada | Timeout muy corto en el cliente | Subir `timeout_seconds` a 180, monitorear primer token |
| macOS reporta "out of memory" | Otros apps consumiendo RAM | Cerrar Chrome, Docker Desktop, IDEs; el agente necesita el headroom |

## 10. Operación continua

- **Persistencia del modelo entre reinicios:** LM Studio recuerda el último modelo cargado. El servidor no se inicia solo, hay que volver a abrir LM Studio y arrancar el server después de cada reboot.
- **Auto-arranque (opcional):** crear un LaunchAgent en `~/Library/LaunchAgents/com.lmstudio.server.plist` que abra LM Studio al login. No habilitar hasta haber validado la PoC manualmente al menos una vez.
- **Actualizaciones del modelo:** LM Studio notifica nuevas cuantizaciones. No actualizar en medio de una corrida activa del agente. Planificar ventanas de actualización.
- **Backups:** copiar `~/.cache/lmstudio/` antes de cualquier cambio mayor de versión de LM Studio. Los modelos son re-descargables, pero la cache de configuracion no.
- **Cuota de disco:** un solo modelo de 5 GB más la cache de LM Studio suman ~7–8 GB. Con 512 GB de disco quedan > 400 GB libres, suficiente para docenas de modelos adicionales. Limpiar modelos no usados desde **My Models → Delete**.

## 11. Checklist de aceptación

Antes de marcar el setup del lab como completo, validar todos los puntos:

- [ ] LM Studio instalado y versión registrada.
- [ ] Modelo Qwen3 8B Instruct Q4_K_M descargado y cargado.
- [ ] Contexto configurado a 8192 tokens.
- [ ] GPU offload al máximo, Flash Attention activado, KV cache en Q8_0.
- [ ] Servidor OpenAI-compatible escuchando en `0.0.0.0:1234`.
- [ ] `curl http://localhost:1234/v1/models` responde con el modelo cargado.
- [ ] `curl http://host.docker.internal:1234/v1/models` desde el contenedor responde igual.
- [ ] Una completion de prueba devuelve respuesta coherente en menos de 10 s.
- [ ] Configuración del agente apunta a `http://host.docker.internal:1234/v1`.
- [ ] Activity Monitor confirma uso de GPU > 0% durante inferencia.
- [ ] Memoria usada por LM Studio estable, sin swapping.
- [ ] Backup de `~/.cache/lmstudio/` realizado.
