# Error #1690130 en WhatsApp Business Coexistence: cómo recuperé los webhooks tras un Embedded Signup interrumpido

**Martín Muñoz · 2 de noviembre de 2026**  
*WhatsApp Cloud API · Embedded Signup · Error 1690130 · Graph API · 131000/10 · Incidente real*

> **Estado real, sin exageraciones:** Meta empezó a mostrar **#1690130** («el identificador comercial no es válido») después de interrumpir un alta de coexistencia, aunque el porfolio seguía existiendo y estaba verificado. **No logré eliminar #1690130 del asistente de Meta.** Sí conseguí **recuperar los mensajes entrantes y los ecos desde WhatsApp Business** sin borrar ni migrar el número, completando la integración pendiente de mi propia app. Sin embargo, **la sincronización del historial y contactos sigue rechazada con 131000, subcódigo 10**. Hay indicios de alta incompleta, pero la causa exacta aún necesita confirmación de Meta.

## Contexto: el segundo error, después de #3441034

Desarrollo una aplicación multiempresa con una bandeja de WhatsApp conectada a la **Cloud API**. Mi número comercial ya funcionaba en la **app de WhatsApp Business**: necesitaba **coexistencia**, no una migración.

Primero sufrí **#3441034**, que dejé de ver tras desconectar **Meta Business Suite Inbox** desde el menú **WhatsApp Business → Ajustes → Cuenta → Plataforma para empresas**. [He publicado ese primer error en un artículo específico](whatsapp-business-error-3441034-solucion-meta-business-suite.md).

Después avancé por el asistente de **Embedded Signup**. Al llegar a la parte de empresa/zona horaria retrocedí y desconecté la plataforma mientras el proceso todavía no había terminado. Cuando traté de retomarlo apareció:

```text
#1690130
[identificador del porfolio] no es un identificador comercial válido
```

El identificador no era inventado. **El porfolio existía y estaba verificado**. Por ello dejé de asumir que el mensaje de la ventana describía literalmente el estado real del recurso.

## Qué NO funcionó

- Reabrir Embedded Signup una y otra vez.
- Repetir el registro desde distintas sesiones.
- Revisar permisos básicos y validar que el porfolio seguía existiendo.
- Volver a desconectar y conectar el dispositivo.
- Buscar una opción para «reparar» el porfolio desde la pantalla del error.

**No recomiendo repetir desconexiones ni eliminar la cuenta.** No comprobé que esos pasos reparasen #1690130 y sí podrían empeorar el estado de una cuenta en uso.

**No se ha demostrado que la zona horaria, la interrupción o que la WABA sea propia sean la causa interna de #1690130**. Son circunstancias del incidente, no un diagnóstico confirmado por Meta.

## Paso 1: averiguar qué había llegado a crear Meta

En lugar de reiniciar el alta, inspeccioné el estado del número, la **WhatsApp Business Account (WABA)** y la suscripción de la aplicación usando **Graph API en modo de lectura**.

Esto fue lo que observé:

| Comprobación | Resultado |
| --- | --- |
| Estado del número | `CONNECTED` |
| Plataforma | `CLOUD_API` |
| Indicador de uso simultáneo de la app | `is_on_biz_app: true` |
| WABA existente | Sí |
| App suscrita a la WABA | **No: `subscribed_apps` estaba vacío** |
| Configuración de campos de webhook | `messages`, `smb_message_echoes`, `history`, `smb_app_state_sync`, `account_update` |
| Canal correspondiente dentro de mi aplicación | **No existía** |
| WhatsApp Manager | En ocasiones mostraba **«Fuera de internet»** |

Consultas ilustrativas, sustituyendo los identificadores por los de tu cuenta y utilizando una versión de Graph admitida:

```http
GET https://graph.facebook.com/vXX.X/<WABA_ID>/phone_numbers
GET https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
GET https://graph.facebook.com/vXX.X/<PHONE_NUMBER_ID>?fields=platform_type,is_on_biz_app
Authorization: Bearer <TOKEN_AUTORIZADO>
```

Los campos que una versión de Graph admite pueden cambiar. Un rechazo de permisos, por ejemplo código 100, no confirma por sí mismo que el WABA no exista.

**Conclusión de la inspección:** el número tenía un estado Cloud API aparentemente operativo, pero faltaban **dos pasos de integración**: suscribir la app a la WABA y guardar el canal en mi software.

Es crucial distinguir **la configuración de los campos del webhook de la app** de la **suscripción de una app a una WABA concreta**. Puedes tener los campos bien configurados y aun así no recibir los webhooks de una WABA no suscrita.

## Paso 2: recuperar solo la parte de la integración que faltaba

Esta es la solución técnica que ejecuté **en mi propia infraestructura y con autorización sobre mi propia WABA**; no es un procedimiento genérico para resolver el error de la ventana de Meta.

1. Creé un **usuario del sistema** dedicado a esa aplicación y a ese WABA.
2. Emití un token con los permisos `whatsapp_business_management` y `whatsapp_business_messaging`.
3. Verifiqué que funcionaba para ese WABA y que no accedía a los cuatro WABA de clientes conocidos con los que probé. Eso es una comprobación limitada, no una prueba de aislamiento sobre todas las WABA del mundo.
4. Guardé el token **en un Vault del servidor**, sin exponerlo al navegador ni registrarlo en logs.
5. Revisé el código que crea canales y probé las precondiciones, el aislamiento y la posibilidad de error parcial **antes de escribir en producción**.
6. Guardé el canal local correspondiente al número ya existente.
7. Suscribí la aplicación a la WABA existente mediante la operación **documentada por Meta**:

```http
POST https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
Authorization: Bearer <TOKEN_DE_USUARIO_DEL_SISTEMA>
```

8. Volví a leer `subscribed_apps` y comprobé que el canal local aparecía activo.

No ejecuté `/<PHONE_NUMBER_ID>/register`; tampoco borré, migré o desregistré el número ni cambié de propietario la WABA. El procedimiento no incluía volver a realizar Embedded Signup.

### Qué se consiguió realmente

El **2 de noviembre de 2026** comprobé lo siguiente:

- La aplicación quedó **suscrita a la WABA**.
- El canal local se creó y quedó **activo**.
- Un **mensaje entrante nuevo** enviado desde otro teléfono apareció tanto en mi bandeja web como en WhatsApp Business (aunque en la tablet el contenido mostraba «Esperando este mensaje»).
- Una **respuesta escrita desde WhatsApp Business** llegó al destinatario y fue registrada por mi sistema como `smb_message_echoes`; también se confirmó un estado `delivered`.

**Eso acredita que los entrantes y los ecos funcionaban.** No demuestra que la tablet descifre correctamente todos los mensajes; tampoco he acreditado todavía el envío independiente desde Cloud API en esa fecha.

## Paso 3: el bloqueo que siguió apareciendo — 131000 / subcódigo 10

Después de recuperar la recepción quise importar el historial previo de WhatsApp Business. Se solicitó la sincronización de contactos y la de conversaciones mediante:

```http
POST https://graph.facebook.com/vXX.X/<PHONE_NUMBER_ID>/smb_app_data
Authorization: Bearer <TOKEN_AUTORIZADO>
```

**Ambas solicitudes fueron rechazadas**, devolviendo:

```text
code: 131000
error_subcode: 10
This API is only available to WhatsApp Business App phone numbers
onboarded on to WhatsApp Business Platform
```

Es un contraste importante: **Graph indicaba `CONNECTED`, `CLOUD_API`, `is_on_biz_app: true` y llegaban webhooks**, pero **`smb_app_data` no aceptaba el número para sincronizar contactos e historial**.

Esto es **compatible con un alta de coexistencia parcialmente completada**, pero no demuestra que esa sea la causa exacta. El mensaje tampoco identificaba un plazo de 24 horas vencido ni una negativa a compartir historial.

Tras esos rechazos, **no repetí la sincronización**. Contacté con la vía de soporte de Meta para pedir una comprobación del onboarding real y de la elegibilidad para historial.

**No recuperé los chats antiguos por API.** El historial del móvil antiguo, la copia de seguridad y el histórico de Cloud API son asuntos diferentes; reinstalar WhatsApp o desregistrar el número para probar puede comprometer la coexistencia y las copias existentes.

## Cómo diagnosticar tu caso sin arriesgar el número

| Síntoma | Primero comprueba | No deduzcas automáticamente |
| --- | --- | --- |
| **#1690130** tras alta interrumpida | Estado real del porfolio, WABA, número y suscripción en Graph | Que el porfolio no existe o haya que eliminarlo |
| `CONNECTED` pero no llegan mensajes | `subscribed_apps`, webhook configurado, persistencia local y pruebas reales | Que el onboarding esté completamente listo |
| Llegan mensajes, pero no los ecos de la app | Suscripción al campo `smb_message_echoes` y eventos recibidos | Que los textos enviados desde el móvil se reflejen solos |
| **131000 / 10** en `smb_app_data` | `fbtrace_id`, fecha, estado Graph y onboarding | Que el historial ya se importó o que el plazo venció |
| **141006** al intentar enviar | Estado de pagos y motivo concreto de Meta | Que la recepción por webhooks tampoco funcione |

## Lo que pediría a Meta Direct Support

1. Confirmar el **estado interno real del onboarding de coexistencia** para el número y explicar **#1690130**.
2. Aclarar por qué `smb_app_data` devuelve **131000 / 10** si `is_on_biz_app: true` y los webhooks y ecos funcionan.
3. Reparar o completar el proceso, **si existe una opción compatible con mantener el número en WhatsApp Business**, sin borrar ni migrar la cuenta.
4. Confirmar expresamente si todavía es posible habilitar la sincronización de contactos/historial después de solicitudes rechazadas.

Envía a Meta los identificadores reales y los `fbtrace_id` **por soporte privado**. No publiques accesos, tokens, teléfonos de clientes ni identificadores de sesión en un artículo abierto.

## Lo que queda sin resolver

**El error #1690130 en Embedded Signup no se ha reparado de manera confirmada.** Lo que se recuperó fue **la integración del canal para mensajes nuevos y ecos**. La sincronización de contactos y conversaciones antiguas seguía bloqueada, y el estado completo de coexistencia requería aclaración de Meta.

Por tanto, este artículo no debe interpretarse como «cómo eliminar #1690130 siempre», sino como **cómo diagnosticar un alta interrumpida, recuperar webhooks cuando los recursos ya existen y reconocer los límites de la recuperación**.

## Referencias técnicas

- [Meta — Embedded Signup y coexistencia de WhatsApp Business](https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users).
- [Meta — Documentación de Embedded Signup (Postman)](https://www.postman.com/meta/whatsapp-business-platform/documentation/du6gzjv/embedded-signup).
- [Meta — Suscribir una app a una WABA (Postman)](https://www.postman.com/meta/whatsapp-business-platform/request/26gui66/subscribe-to-a-waba).
- [Bird — Errores y diagnóstico de coexistencia](https://bird.com/en-nl/docs/knowledge-base/whatsapp/connection-errors).
- [Primer error: solución que me funcionó para #3441034](whatsapp-business-error-3441034-solucion-meta-business-suite.md).

---

**Martín Muñoz** · [GitHub personal](https://github.com/martinmunozb) · Caso documentado el **2 de noviembre de 2026**. Último estado: recepción y eco confirmados; sincronización histórica bloqueada.

Si te ocurrió lo mismo, puedes compartir el **código**, el **paso donde falla Embedded Signup** y el resultado anonimizado de `subscribed_apps`. Ese contexto es mucho más útil que repetir el registro a ciegas.
