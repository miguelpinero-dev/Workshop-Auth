# Workshop-Auth
Workshop sobre autenticación en APIs: login, JWT, access &amp; refresh tokens, flujos y vulnerabilidades, enfocado en iOS.

# Prompt 1 – Backend inseguro (caso base)

Este primer prompt genera **un backend intencionalmente inseguro**, que vamos a usar como punto de partida para entender **qué NO hacer** y poder testear vulnerabilidades comunes en flujos de autenticación.

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para generar el backend:

```text
Creá un backend simple en Node.js con Express (JavaScript, no TypeScript) que corra en puerto 3000.
Implementá POST /login con credenciales hardcodeadas (test@demo.com / 123456) y devolvé un JWT firmado con un secret fijo en el código.
El JWT debe NO tener expiración (sin exp).
Implementá GET /profile protegido por middleware que valide el JWT desde Authorization: Bearer <token> y devuelva { userId, email } usando SOLO el payload del token (no DB).
Agregá POST /refresh que reciba { refreshToken } y devuelva un nuevo access token; el refresh token debe guardarse en memoria y debe poder reutilizarse infinitas veces (sin rotación).
No agregues rate limiting en /login.
Los errores pueden ser detallados (ej: token expirado vs inválido vs faltante).
Incluí un endpoint GET /health.
A partir de ahora, si cambiás index.js, tenés que reiniciar el servidor para que use el código nuevo.
Encargate vos de hacer el npm start del principio
```
---

## Pruebas con curl – Prompt 1

A continuación se muestran distintos comandos `curl` para testear el comportamiento del backend generado con el **Prompt 1** y evidenciar sus vulnerabilidades.

---

### 🔐 Token eterno (un login = token para siempre)

El access token no tiene expiración, por lo que un solo login alcanza para usar la API indefinidamente.

```bash
# Login una sola vez.
# El backend devuelve un access token SIN expiración.
# Guardamos el token en un archivo temporal.
curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | jq -r '.accessToken' > /tmp/token.txt

# Usamos el mismo token en /profile.
# Como el token no expira, este request funcionará "para siempre".
curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer $(cat /tmp/token.txt)"
```

### 🧪 Token inválido 

```bash
# Intentamos acceder a /profile con un token cualquiera.
curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer cualquier-token"
```

### 🧠 Confianza en el payload (sin validar contra DB)

```bash
# El backend confía ciegamente en el contenido del token.
# No hay validación contra base de datos.
curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer $(cat /tmp/token.txt)"
```

### 🔁 Refresh token reutilizable (sin rotación)

```bash
# Obtenemos el refresh token desde el login.
REFRESH=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | jq -r '.refreshToken')

# Usamos el mismo refresh token varias veces.
# El backend permite su reutilización infinita.
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}"

curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}"

curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}"
```

### 🚨 Sin rate limit en /login

```bash
# Simulamos múltiples intentos de login seguidos (brute force).
# El backend no aplica ningún rate limit.
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code} " \
    -X POST http://localhost:3000/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@demo.com","password":"wrong"}'
done
echo ""
```

# Prompt 2 – Manejo consistente de Authorization (401 genérico)

En este segundo prompt **no agregamos nuevas features**, sino que ajustamos el comportamiento de seguridad de los endpoints protegidos para que **no filtren información** a un posible atacante.

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para modificar el backend existente:

```text
En este proyecto, asegurá que los endpoints protegidos respondan consistentemente:
GET /profile debe devolver 401 si falta Authorization o si el token no es válido.
Exigí estrictamente el formato Authorization: Bearer <token>.
Por seguridad, no especifiques el tipo de error (token faltante, expirado, inválido, formato incorrecto, etc.): devolvé siempre el mismo mensaje y/o código genérico (por ejemplo "Unauthorized" o "No autorizado") para no filtrar información a un atacante.
No cambies nada más (no agregues expiración, no refresh nuevo, no DB, no rate limiting).
Solo ajustá validación de header y status codes.
Reinicia el servidor y corre uno nuevo para verificar estos nuevos cambios.
Al hacer cambios en el código del proyecto, reiniciá el servidor (kill + npm start) y encargate vos de hacerlo.
```
---

## Pruebas con curl – Prompt 2

A continuación se prueban distintos escenarios para verificar que el backend responde siempre de forma consistente con 401, sin filtrar detalles del error.
---

### 🚫 Sin header Authorization

```bash
# No enviamos el header Authorization.
# El backend debe responder 401.
curl -s -w "\n%{http_code}" http://localhost:3000/profile
```

### ⚠️ Formato inválido (solo "Bearer", sin token)

```bash
# Enviamos Authorization sin token.
# El backend debe responder 401, sin detallar el error.
curl -s -w "\n%{http_code}" http://localhost:3000/profile \
  -H "Authorization: Bearer"
```

### ❌ Token inválido

```bash
# Enviamos un token inválido.
# El backend debe responder 401 genérico.
curl -s -w "\n%{http_code}" http://localhost:3000/profile \
  -H "Authorization: Bearer token.invalido"
```

### ✅ Token válido (comprobar que sigue funcionando)

```bash
# Obtenemos un access token válido desde /login.
TOKEN=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

# Usamos el token válido en /profile.
# El endpoint debe responder 200.
curl -s -w "\n%{http_code}" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

# Prompt 3 – Expiración del access token (1 minuto) + 401 por expirado

En este prompt agregamos **solo** expiración al access token para poder ver el flujo real de “token vencido” y cómo el backend responde con **401**.

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para modificar el backend existente:

```text
Modificá únicamente la emisión del JWT de acceso en POST /login para que expire en 1 minuto (expiresIn: '1m') e incluya exp.
Ajustá el middleware para que tokens expirados también devuelvan 401.
No implementes refresh tokens distintos ni rotación, no agregues DB ni rate limiting.
Solo expiración de access token y manejo 401.
Al hacer cambios en el código del proyecto, reiniciá el servidor (kill + npm start) y encargate vos de hacerlo.
```
---

## Pruebas con curl – Prompt 3

Objetivo: confirmar que el token funciona al inicio (200) y que luego de 1 minuto expira (401).
---

### ⚠️ Login + Probar profile (status 200) + 60 seg + Profile (error 401)

```bash
# 1) Hacemos login y obtenemos un access token que expira en 1 minuto.
# 2) Probamos /profile inmediatamente (debe dar 200).
# 3) Esperamos 60 segundos.
# 4) Probamos /profile de nuevo (debe dar 401 por expirado).
TOKEN=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken"); \
echo "Profile ahora (debe ser 200):"; \
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"; \
echo "Esperando 60 segundos..."; \
sleep 60; \
echo "Profile después de 1 min (debe ser 401):"; \
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

# Prompt 4 – Validación contra DB (SQLite) + usuario activo/inactivo

En este prompt agregamos una base local SQLite para que el backend no confíe solo en el JWT.  
El endpoint `/profile` debe validar el token **y además** verificar en DB que el usuario exista y esté activo.

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para modificar el backend existente:

```text
Agregá SQLite local (archivo data.db) usando el paquete better-sqlite3 con tabla users(id INTEGER PRIMARY KEY, email TEXT UNIQUE, is_active INTEGER).
Al iniciar el server, crear la tabla si no existe e insertar/asegurar el usuario id=1, email=test@demo.com, is_active=1.
Modificá GET /profile para que, además de validar el JWT, consulte el usuario por id=sub y solo responda si existe y is_active=1; si no, responder 401.
No cambies login/refresh aún salvo lo mínimo para que compile.
Al hacer cambios en el código del proyecto, reiniciá el servidor (kill + npm start) y encargate vos de hacerlo.
```
---

## Pruebas con curl – Prompt 4
---

### ✅ Login + profile (usuario existe y está activo)

```bash
# Obtenemos un access token válido.
TOKEN=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

# Con el usuario activo en DB, /profile debe responder 200.
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

### 🚫 Usuario desactivado en DB → 401

```bash
# Desactivamos el usuario en la DB (ejecutar en la carpeta del proyecto).
node -e "
const Database = require('better-sqlite3');
const db = new Database('data.db');
db.prepare('UPDATE users SET is_active = 0 WHERE id = 1').run();
console.log('Usuario id=1 desactivado.');
db.close();
"

# Obtenemos un token (login sigue permitido porque login aún no usa DB).
TOKEN=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

# Como el usuario está inactivo en DB, /profile debe responder 401.
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

### ✅ Reactivar y ver que vuelve a dar 200

```bash
# Reactivamos el usuario en la DB.
node -e "
const Database = require('better-sqlite3');
const db = new Database('data.db');
db.prepare('UPDATE users SET is_active = 1 WHERE id = 1').run();
console.log('Usuario id=1 reactivado.');
db.close();
"

# Reusamos el mismo token de antes (no hace falta volver a hacer login).
# Con el usuario activo nuevamente, /profile debe responder 200.
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

# Prompt 5 – Access + Refresh tokens (expiración + persistencia en DB)

En este prompt pasamos a un flujo más realista: `accessToken` de vida corta y `refreshToken` de vida un poco mayor, con refresh tokens persistidos en SQLite.  
Todavía **no** agregamos rotación de refresh ni rate limiting: solo persistencia, expiración y validaciones correctas.

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para modificar el backend existente:

```text
Convertí el flujo a accessToken + refreshToken: POST /login debe devolver únicamente { accessToken, refreshToken } (sin otros campos como user), donde accessToken expira 1m y refreshToken expira 3 minutos.
Guardá refresh tokens en SQLite en tabla refresh_tokens(token TEXT PRIMARY KEY, user_id INTEGER, expires_at TEXT).
Implementá POST /refresh que reciba { refreshToken }, valide que exista, no esté expirado y que el usuario asociado exista y esté activo, y devuelva un nuevo accessToken (por ahora sin rotación del refresh).
No agregues rate limiting todavía.
Al hacer cambios en el código del proyecto, reiniciá el servidor (kill + npm start) y encargate vos de hacerlo.
```
---

## Pruebas con curl – Prompt 5
---

### ✅ Login: solo accessToken y refreshToken

```bash
# Verificar que /login devuelve únicamente { accessToken, refreshToken }.
curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}'
```

### ✅ Profile con accessToken y refresh para obtener nuevo access

```bash
# Login y guardar tokens.
RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

ACCESS=$(echo "$RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")
REFRESH=$(echo "$RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).refreshToken")

# Profile con accessToken (debe dar 200 mientras no expire).
echo "Profile con accessToken:"
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $ACCESS"

# Nuevo accessToken vía refresh (sin rotación: mismo refreshToken).
echo "Refresh:"
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}"

# Opcional: obtener un nuevo access token y probarlo en /profile.
NEW_ACCESS=$(curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}" \
  | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo "Profile con nuevo accessToken:"
curl -s -w "\n%{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $NEW_ACCESS"
```

### ⏳ Refresh token expirado (después de 3 min)

```bash
# Usar el mismo REFRESH del bloque anterior.
# Esperar 3 minutos y luego ejecutar:
curl -s -w "\n%{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}"
```

### ❌ Refresh token inválido o faltante

```bash
# Token inexistente (no está en la tabla refresh_tokens).
# El backend debe responder 401.
curl -s -w "\n%{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"token-que-no-existe"}'

# Body sin refreshToken.
# El backend debe responder 401.
curl -s -w "\n%{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d '{}'
```

# Prompt 6 – Rotación de refresh tokens + rate limiting en /login (429)

En este prompt agregamos **dos medidas concretas** y nada más:
1) **Rotación de refresh tokens** (el refresh se usa una sola vez).  
2) **Rate limiting** solo en `POST /login` (máximo 5 intentos por minuto por IP).

---

## 🧠 Prompt

Usar este prompt en Cursor / ChatGPT para modificar el backend existente:

```text
Implementá dos cosas y nada más: (1) Rotación de refresh tokens: en POST /refresh, cuando el refresh sea válido, eliminá el refresh usado de la DB, generá uno nuevo con expiración 3 minutos, guardalo y devolvé { accessToken, refreshToken }; si se intenta reutilizar un refresh ya usado/no existente, responder 401. (2) Agregá rate limiting SOLO en POST /login: máximo 5 intentos por minuto por IP, y si excede responder 429 { error: 'too_many_requests' }. No agregues otras medidas extra. Al hacer cambios en el código del proyecto, reiniciá el servidor (kill + npm start) y encargate vos de hacerlo.
```
---

## Pruebas con curl – Prompt 6
---

### 🔁 Rotación: el refresh devuelve uno nuevo y el viejo deja de servir

```bash
# Login y obtener refresh token inicial (REFRESH1).
RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

REFRESH1=$(echo "$RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).refreshToken")

# Primer /refresh con REFRESH1:
# - REFRESH1 se invalida (se borra de DB)
# - se genera REFRESH2 (nuevo)
RES_FIRST=$(curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH1\"}")

REFRESH2=$(echo "$RES_FIRST" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).refreshToken")

# Reutilizar REFRESH1 (ya usado) -> debe dar 401.
curl -s -w "\n%{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH1\"}"

# Usar REFRESH2 (el nuevo) -> debe funcionar y devolver un nuevo par { accessToken, refreshToken }.
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH2\"}"
```

### 🚦 Rate limit en POST /login: 6.º intento → 429

```bash
# 6 intentos de login en menos de 1 minuto (pueden ser con password incorrecto).
# Los primeros 5 deberían responder normalmente (ej. 401).
# El intento 6 debería responder 429 con { error: "too_many_requests" }.
for i in 1 2 3 4 5 6; do
  echo "Intento $i:"
  curl -s -w " -> %{http_code}\n" -X POST http://localhost:3000/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@demo.com","password":"wrong"}'
done

```







