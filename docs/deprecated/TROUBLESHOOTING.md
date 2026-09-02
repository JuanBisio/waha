# Solución al Problema del QR Code

## Problema Identificado

Docker no está corriendo en tu sistema. Docker.app está instalado pero necesitas iniciarlo.

## Solución Rápida

### 1. Iniciar Docker Desktop

```bash
# Abrir Docker Desktop
open /Applications/Docker.app
```

**Espera 30-60 segundos** hasta que veas el ícono de Docker en la barra superior de macOS y diga "Docker Desktop is running".

### 2. Verificar que Docker está corriendo

```bash
docker ps
```

Si funciona, verás una lista de contenedores (puede estar vacía).

### 3. Iniciar Evolution API

```bash
cd /Users/bisiojuan/Desktop/joacoJuan
docker compose up -d
```

### 4. Verificar contenedores

```bash
docker compose ps
```

Deberías ver 3 contenedores: `evolution_api`, `evolution_postgres`, `evolution_redis`

### 5. Ver logs de Evolution API

```bash
docker compose logs -f evolution-api
```

Presiona `Ctrl+C` para salir de los logs.

### 6. Intentar crear instancia nuevamente

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -d '{
    "instanceName": "SaaS_Instance",
    "qrcode": true,
    "integration": "WHATSAPP-BAILEYS"
  }'
```

### 7. Obtener QR Code

**Opción A: Via API**
```bash
curl -X GET http://localhost:8080/instance/connect/SaaS_Instance \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
```

**Opción B: Via Manager UI (MÁS FÁCIL)**
1. Abre en tu navegador: http://localhost:8080/manager
2. Ingresa el API Key cuando te lo pida: `SaaS_Instance_Key_2026_Secure_Token_12345`
3. El QR code debería aparecer automáticamente en la interfaz

## Alternativa: Si Docker te da problemas

### Opción Más Simple: Usar CallMeBot API

Si Docker es muy complicado, puedo configurarte un sistema más simple usando CallMeBot que no requiere Docker:

**Ventajas:**
- ✅ No requiere Docker
- ✅ Configuración en 2 minutos
- ✅ Gratis para uso personal
- ✅ Solo necesitas tu número de WhatsApp

**Desventaja:**
- ⚠️ Tienes que agregar un contacto a tu WhatsApp

¿Quieres que te configure esta alternativa?

## Comandos Útiles de Docker

```bash
# Iniciar Docker Desktop
open /Applications/Docker.app

# Ver contenedores corriendo
docker ps

# Ver todos los contenedores
docker ps -a

# Iniciar servicios
docker compose up -d

# Detener servicios
docker compose down

# Ver logs
docker compose logs -f evolution-api

# Reiniciar un servicio específico
docker compose restart evolution-api

# Ver estado de salud
docker compose ps
```

## Si el problema persiste

Si después de iniciar Docker Desktop sigues teniendo problemas:

1. **Verifica que el puerto 8080 no esté ocupado:**
```bash
lsof -i :8080
```

2. **Elimina contenedores antiguos y vuelve a empezar:**
```bash
docker compose down -v
docker compose up -d
```

3. **Revisa los logs para ver errores específicos:**
```bash
docker compose logs evolution-api
```

Avísame qué método prefieres y te ayudo a configurarlo! 🚀
