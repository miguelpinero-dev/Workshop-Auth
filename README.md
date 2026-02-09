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
echo -e "\n🔐 LOGIN – obteniendo accessToken"
echo "----------------------------------------"

curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | jq -r '.accessToken' | tee /tmp/token.txt

echo -e "\n📄 PROFILE – usando el mismo token"
echo "----------------------------------------"

curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer $(cat /tmp/token.txt)" | jq .
```

### 🧪 Token inválido 

```bash
echo -e "\n🚫 PROFILE – token inválido"
echo "----------------------------------------"

curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer cualquier-token" \
  | jq . 2>/dev/null || echo "Respuesta no JSON"
```

### 🧠 Confianza en el payload (sin validar contra DB)

```bash
echo -e "\n🧠 PROFILE – confianza en el payload del token (sin DB)"
echo "----------------------------------------"

curl -s http://localhost:3000/profile \
  -H "Authorization: Bearer $(cat /tmp/token.txt)" \
  | jq . 2>/dev/null || echo "Respuesta no JSON"
```

### 🔁 Refresh token reutilizable (sin rotación)

```bash
echo -e "\n🔁 LOGIN – obteniendo refreshToken"
echo "----------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

REFRESH=$(echo "$LOGIN_RES" | jq -r '.refreshToken')

echo "refreshToken:"
echo "$REFRESH"

echo -e "\n♻️ REFRESH #1"
echo "----------------------------------------"
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH" '{refreshToken: $rt}')"
```

```bash

echo -e "\n♻️ REFRESH #2 (mismo refreshToken)"
echo "----------------------------------------"
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH" '{refreshToken: $rt}')"
```

```bash
echo -e "\n♻️ REFRESH #3 (mismo refreshToken)"
echo "----------------------------------------"
curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH" '{refreshToken: $rt}')"
```

### 🚨 Sin rate limit en /login

```bash
echo -e "\n🚨 BRUTE FORCE – múltiples intentos de login (sin rate limit)"
echo "-----------------------------------------------------------"

for i in $(seq 1 20); do
  echo -n "Intento $i -> "
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:3000/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@demo.com","password":"wrong"}'
done
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
echo -e "\n🚫 PROFILE – sin header Authorization"
echo "----------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile
```

### ⚠️ Formato inválido (solo "Bearer", sin token)

```bash
echo -e "\n⚠️ PROFILE – Authorization con formato inválido (Bearer sin token)"
echo "---------------------------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer"
```

### ❌ Token inválido

```bash
echo -e "\n❌ PROFILE – token inválido"
echo "----------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer token.invalido"
```

### ✅ Token válido (comprobar que sigue funcionando)

```bash
echo -e "\n✅ LOGIN – obteniendo accessToken válido"
echo "----------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

TOKEN=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo "accessToken:"
echo "$TOKEN"

echo -e "\n✅ PROFILE – usando token válido"
echo "----------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN" \
  | jq . 2>/dev/null || true
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
echo -e "\n⏱️ LOGIN – accessToken con expiración de 1 minuto"
echo "-----------------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

TOKEN=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo "accessToken:"
echo "$TOKEN"

echo -e "\n✅ PROFILE inmediato (debe ser 200)"
echo "----------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"

echo -e "\n⏳ Esperando 60 segundos..."
sleep 60

echo -e "\n❌ PROFILE después de 1 minuto (debe ser 401)"
echo "--------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
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
echo -e "\n🟢 LOGIN – obteniendo accessToken (usuario activo en DB)"
echo "-------------------------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

TOKEN=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo -e "\naccessToken:"
echo "$TOKEN"

echo -e "\n✅ PROFILE – usuario activo en DB (debe ser 200)"
echo "-----------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN" \
  | jq . 2>/dev/null || true
```

### 🚫 Usuario desactivado en DB → 401

```bash
echo -e "\n🚫 DB – desactivando usuario (id=1)"
echo "----------------------------------"

node -e "
const Database = require('better-sqlite3');
const db = new Database('data.db');
db.prepare('UPDATE users SET is_active = 0 WHERE id = 1').run();
console.log('Usuario id=1 desactivado.');
db.close();
"

echo -e "\n🔐 LOGIN – obteniendo accessToken (login sigue permitido)"
echo "--------------------------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

TOKEN=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo -e "\n❌ PROFILE – usuario INACTIVO en DB (debe ser 401)"
echo "-------------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN"
```

### ✅ Reactivar y ver que vuelve a dar 200

```bash
echo -e "\n🟢 DB – reactivando usuario (id=1)"
echo "----------------------------------"

node -e "
const Database = require('better-sqlite3');
const db = new Database('data.db');
db.prepare('UPDATE users SET is_active = 1 WHERE id = 1').run();
console.log('Usuario id=1 reactivado.');
db.close();
"

echo -e "\n✅ PROFILE – usuario ACTIVO nuevamente (mismo token)"
echo "----------------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $TOKEN" \
  | jq . 2>/dev/null || true
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
echo -e "\n🔐 LOGIN – devuelve solo accessToken y refreshToken"
echo "-----------------------------------------------"

curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}' \
  | jq .

```

### ✅ Profile con accessToken y refresh para obtener nuevo access

```bash
echo -e "\n🔐 LOGIN – obteniendo accessToken y refreshToken"
echo "----------------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

ACCESS=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")
REFRESH=$(echo "$LOGIN_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).refreshToken")

echo -e "\naccessToken:"
echo "$ACCESS"
echo -e "\nrefreshToken:"
echo "$REFRESH"

echo -e "\n✅ PROFILE – usando accessToken (debe ser 200)"
echo "---------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $ACCESS"

echo -e "\n🔁 REFRESH – obteniendo nuevo accessToken (sin rotación)"
echo "-------------------------------------------------------"

REFRESH_RES=$(curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH" '{refreshToken: $rt}')")

echo "$REFRESH_RES" | jq .

NEW_ACCESS=$(echo "$REFRESH_RES" | node -p "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).accessToken")

echo -e "\n✅ PROFILE – usando nuevo accessToken"
echo "------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" http://localhost:3000/profile \
  -H "Authorization: Bearer $NEW_ACCESS"
```

### ⏳ Refresh token expirado (después de 3 min)

```bash
echo -e "\n⏳ REFRESH – esperando expiración del refreshToken (3 minutos)"
echo "-------------------------------------------------------------"

echo "Esperando 3 minutos..."
sleep 180

echo -e "\n❌ REFRESH – refreshToken expirado (debe ser 401)"
echo "-------------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH" '{refreshToken: $rt}')"
```

### ❌ Refresh token inválido o faltante

```bash
echo -e "\n❌ REFRESH – token inexistente (no está en DB)"
echo "---------------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"token-que-no-existe"}'


echo -e "\n❌ REFRESH – body sin refreshToken"
echo "--------------------------------"

curl -s -w "\nStatus: %{http_code}\n" -X POST http://localhost:3000/refresh \
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
echo -e "\n🔐 LOGIN – obteniendo refreshToken inicial (REFRESH1)"
echo "--------------------------------------------------"

LOGIN_RES=$(curl -s -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@demo.com","password":"123456"}')

REFRESH1=$(echo "$LOGIN_RES" | jq -r '.refreshToken')

echo -e "\nREFRESH1:"
echo "$REFRESH1"
```
```bash
echo -e "\n🔁 REFRESH #1 – rotación de refreshToken"
echo "---------------------------------------"

REFRESH_RES_1=$(curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH1" '{refreshToken: $rt}')")

echo "$REFRESH_RES_1" | jq .

REFRESH2=$(echo "$REFRESH_RES_1" | jq -r '.refreshToken')

echo -e "\nREFRESH2 (nuevo):"
echo "$REFRESH2"

```
```bash
echo -e "\n🚫 REUTILIZAR REFRESH1 – debe dar 401"
echo "------------------------------------"

curl -s -w "\nStatus: %{http_code}\n" -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH1" '{refreshToken: $rt}')"

```
```bash
echo -e "\n✅ USAR REFRESH2 – debe funcionar y rotar nuevamente"
echo "----------------------------------------------------"

curl -s -X POST http://localhost:3000/refresh \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg rt "$REFRESH2" '{refreshToken: $rt}')"
```

### 🚦 Rate limit en POST /login: 6.º intento → 429

```bash
echo -e "\n🚦 RATE LIMIT – POST /login (5 intentos por minuto)"
echo "-----------------------------------------------"

for i in 1 2 3 4 5 6; do
  echo -n "Intento $i -> "
  curl -s -w "%{http_code}\n" -X POST http://localhost:3000/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@demo.com","password":"wrong"}'
done
```







