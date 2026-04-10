🛡️ Reporte de Mitigación de Vulnerabilidades - Certifix
Nivel de Riesgo Global: 🚨 CRÍTICO

1. Vulnerabilidades Críticas (Prioridad 0 - ¡Arreglar YA!)
A. Fuga Total de PII y Datos Multi-tenant (RLS Rotas)
Problema: Las políticas de seguridad (RLS) en appointments, shops, services, etc., están configuradas como true. Esto significa que la Anon Key (que es pública) permite a cualquiera leer e insertar datos en cualquier tienda.

Impacto: Exposición masiva de nombres, correos y teléfonos de clientes.

Solución: * Eliminar la migración 1001_rollback_rls.sql.

Implementar políticas que validen el shop_id. Solo el rol service_role o usuarios autenticados con permisos específicos deben ver estos datos.

B. Tenencia sin Protección (RLS Deshabilitado)

Problema: La migración 9000_disable_rls_debug.sql apagó la seguridad en las tablas maestras shops y shop_users.

Impacto: Cualquiera puede cambiarse de tienda o hacerse "dueño" de una tienda ajena.

Solución: Borrar ese archivo de migración inmediatamente y ejecutar ALTER TABLE shops ENABLE ROW LEVEL SECURITY;.

2. Vulnerabilidades de Nivel Alto (Prioridad 1)
A. Endpoint Público /api/public/appointments con Superpoderes

Problema: El endpoint usa la service_role key, que ignora toda seguridad de la base de datos, y no tiene validaciones de coherencia ni límites de velocidad (Rate Limit).

Impacto: Un bot podría llenar tu base de datos de turnos basura en segundos (Spam masivo) o corromper las relaciones entre servicios y profesionales.

Solución: * Cambiar a la anon key para este endpoint.

Añadir validaciones: verificar que el service_id realmente pertenezca al shop_id antes de insertar.

B. Test de Webhooks sin Autenticación (SSRF)

Problema: Cualquier persona puede enviar peticiones desde tu servidor a URLs externas usando el endpoint de prueba de webhooks.

Impacto: Uso de tu infraestructura para atacar a terceros o escanear redes internas.

Solución: Restringir el acceso a este endpoint solo para usuarios con rol admin o owner de la tienda.

3. Vulnerabilidades de Nivel Medio (Prioridad 2)
A. Secretos Hardcodeados (Tokens Predecibles)

Problema: Si fallan las variables de entorno, el sistema usa valores fijos por defecto para firmar webhooks y cancelaciones.

Impacto: Un atacante puede predecir tokens de cancelación y anular turnos de forma masiva.

Solución: Eliminar los fallbacks de texto plano. El código debe lanzar un error si WEBHOOK_MASTER_SECRET no está configurado.

B. Rate-Limit Local (In-Memory)

Problema: El límite de 100 peticiones por minuto se guarda en la memoria de la instancia.

Impacto: Al estar en n8n o servicios cloud que escalan, cada instancia tiene su propio contador, permitiendo saltarse el límite fácilmente.

Solución: Implementar un Rate Limiter centralizado usando Redis (ej: Upstash) para que el conteo sea global.

🚀 Plan de Acción - Siguientes Pasos
Limpieza de Migraciones: Borrá los archivos 9000_... y 1001_... de tu carpeta de migraciones.

Refuerzo de RLS: Ejecutá un script SQL que habilite RLS en todas las tablas y cree políticas de SELECT basadas en autenticación.

Middleware de Seguridad: Añadir un middleware en Next.js que bloquee el acceso a rutas /api/v1/admin/* si no hay una sesión de Supabase válida.

¿Querés que te redacte el código SQL exacto para las nuevas políticas de RLS que protejan la tabla appointments asegurando que nadie pueda leer turnos de otros negocios? Podés correrlo directo en el editor de Supabase.