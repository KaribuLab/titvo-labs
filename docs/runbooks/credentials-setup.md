# Runbook — Configuración de credenciales autenticadas

## 1. Propósito

Habilitar al agente `pentest-agent` para correr con una o más identidades autenticadas contra el objetivo autorizado, sin que ningún valor de credencial toque el filesystem del lab.

**Audiencia:** operador del lab.
**Resultado esperado:** `pentest-agent config-check` valida `auth:` correctamente y `pentest-agent run` levanta sesiones para cada identidad declarada.

## 2. Patrón de dos archivos

El agente usa dos archivos separados:

| Archivo | Versionado | Contenido |
|---|---|---|
| `config/config.yaml` | sí | bloque `auth:` con `credentials_file`, `identity`, `login_endpoint`, `flow` |
| `config/credentials.yaml` | **no** (en `.gitignore`) | lista de `{id, username, password_env}` o `{id, token_env}` (solo nombres de env vars) |

El valor de la credencial vive **solo** en `os.environ` del proceso que ejecuta el agente. Rotación = cambiar la env var en el shell.

## 3. Procedimiento

### 3.1 Copiar el ejemplo

```bash
cd pentesting
cp config/credentials.yaml.example config/credentials.yaml
```

`credentials.yaml.example` no contiene secretos: solo nombres de env vars como `PASSWORD_USER_REGULAR`, `PASSWORD_USER_ADMIN`, `SERVICE_BEARER_TOKEN`.

### 3.2 Definir las env vars en el shell

Opción A — directo en el shell (sesión actual):

```bash
export PASSWORD_USER_REGULAR="..."
export PASSWORD_USER_ADMIN="..."
export SERVICE_BEARER_TOKEN="..."
```

Opción B — archivo `pentest.env` (recomendado, versionable sin secretos porque el archivo es local):

```bash
cat > /tmp/pentest.env <<'EOF'
PASSWORD_USER_REGULAR=...
PASSWORD_USER_ADMIN=...
SERVICE_BEARER_TOKEN=...
EOF
chmod 600 /tmp/pentest.env
```

Opción C — `direnv` con un `.envrc` local:

```bash
cat > pentesting/.envrc <<'EOF'
export PASSWORD_USER_REGULAR="..."
export PASSWORD_USER_ADMIN="..."
EOF
echo "pentesting/.envrc" >> .gitignore  # no versionar
direnv allow pentesting/
```

### 3.3 Validar antes de correr

```bash
docker compose run --rm \
  --env-file /tmp/pentest.env \
  pentest-agent config-check
```

Salida esperada cuando todo está bien:

```
[ok] provider: lmstudio
     model: qwen/qwen3-8b
     base_url: http://host.docker.internal:1234/v1
[ok] credentials file: /etc/pentest/credentials.yaml
[ok] auth identities: user_regular, user_admin
[ok] login_endpoint: https://app.example.internal/api/login
[ok] flow: json
```

Si falta una env var, el comando imprime los nombres exactos y sale con `exit 1`. Solucionarlo antes de continuar.

### 3.4 Lanzar la ejecución

```bash
docker compose run --rm \
  --env-file /tmp/pentest.env \
  pentest-agent run \
    --target app.example.internal \
    --config /etc/pentest/config.yaml
```

Durante el ciclo, el agente ejecuta la fase `AUTENTICAR` antes de la primera hipótesis autenticada: para cada `identity` declarada, llama al `login_endpoint` con las credenciales resueltas desde las env vars y persiste una `Session` en SQLite.

## 4. Rotación de una credencial

1. Cambiar el valor en el shell (o en `pentest.env`): `export PASSWORD_USER_REGULAR="nuevo-valor"`.
2. Re-correr `config-check` para confirmar.
3. Re-correr `pentest-agent run`.

No hace falta tocar `credentials.yaml`, ni reinstalar la imagen, ni reiniciar LM Studio.

## 5. Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `[FAIL] credentials file: not found` | `auth.credentials_file` apunta a un path inexistente | Verificar el path en `config.yaml`; suele ser relativo a `config/` |
| `[FAIL] env vars faltantes: user_regular (env=PASSWORD_USER_REGULAR)` | La env var no está en el proceso | Agregarla al shell o al `--env-file` |
| `auth.identity declarado pero credentials_file falta` | `config.yaml` tiene `auth.identity` sin `auth.credentials_file` | Agregar la referencia |
| `pentest-agent run` falla con `AuthFlowError: login endpoint ... returned 401` | Las credenciales no son válidas contra el target | Confirmar que el target espera el formato (JSON `{username, password}` vs HTML form) y cambiar `auth.flow` si hace falta |
| `ReplayCredentialError: missing env vars for credentials: cred-alice (env=...)` durante `replay` | La credencial cambió o se rotó desde la ejecución original | Re-rotar la env var al valor usado en la ejecución original; **o** aceptar que el replay no puede re-autenticar |

## 6. Modelo de seguridad — qué NO hacer

- **No** commitear `credentials.yaml` con valores. El archivo está en `.gitignore`; respetalo.
- **No** pasar credenciales por stdin en CI/CD sin rotar después: deja una ventana de exposición.
- **No** usar `password_env: $(cat secrets.txt)`: el archivo queda en el historial del shell.
- **No** declarar `auth.identity: ["*"]`: el agente valida que cada label exista en `credentials.yaml`, así que no es un riesgo directo, pero limita la trazabilidad.

## 7. Próximo paso (v2, diferido)

Un MCP server que reemplace `credentials_file` con un backing store real (Vault, AWS Secrets Manager, Doppler) sin tocar el resto del agente. La interfaz del lado del agente (`auth.credentials_file` → `list[CredentialSpec]`) queda igual; solo cambia el resolver.

Para v1 el patrón de env vars es suficiente y mantiene el agente ejecutable en una sola máquina, sin servicios externos.