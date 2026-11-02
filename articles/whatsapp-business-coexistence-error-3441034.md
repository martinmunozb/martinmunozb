# Cómo recuperé WhatsApp Business Coexistence: errores #3441034, #2494064 y #1690130

**Martín Muñoz · 2 de noviembre de 2026**  
**WhatsApp Business · Meta Cloud API · Embedded Signup · Coexistence · Debugging**

> **Resumen:** Si aparece **#3441034** al intentar conectar un número que ya usas en WhatsApp Business, revisa si está vinculado a **Meta Business Suite Inbox** en el móvil. Eso bloqueaba mi conexión. Después sufrí un alta interrumpida y un **#1690130**. En ese segundo caso, Meta ya mostraba `CONNECTED` y `is_on_biz_app: true`, pero mi aplicación no estaba suscrita al WABA ni tenía guardado el canal. Pude recuperar **la recepción de mensajes** sin volver a registrar el número.

Este artículo cuenta **un caso real**, no una solución universal. Escribo en primera persona porque fui siguiendo los errores y comprobando cada estado. No trabajo para Meta y no tengo acceso a sus diagnósticos internos.

## El escenario

Desarrollo una aplicación que conecta WhatsApp Business con la **WhatsApp Cloud API** mediante **Embedded Signup con coexistencia**. El requisito era mantener el mismo número operativo en WhatsApp Business (móvil/tablet) y recibir mensajes también en la bandeja web.

El flujo se detuvo dos veces por **problemas diferentes**:

| Síntoma o código | Qué observé | Qué funcionó o qué comprobar |
| --- | --- | --- |
| **#3441034** — «No cumple los requisitos para el registro ni la migración» | El número ya estaba conectado a **Meta Business Suite Inbox** | Desvincular **solo esa plataforma** desde WhatsApp Business; después pude avanzar |
| **#2494064** — «El número ya está registrado» | Lo probé en otro contexto de registro | Volver al flujo de **coexistencia**; no borrar el número |
| **#1690130** — «[ID de empresa] no es un identificador comercial válido» | Apareció tras interrumpir el alta a la altura del porfolio/zona horaria | **Causa exacta no confirmada**; la recuperación final se hizo comprobando Graph y terminando la integración pendiente |
| «Fuera de internet» en WhatsApp Manager | Graph mostraba el número como `CONNECTED` | No interpretar un solo indicador como diagnóstico definitivo: verificar webhooks y mensajes reales |
| Plataforma aparece «conectada», pero la bandeja está vacía | El número estaba en Meta, pero no había canal local ni suscripción del WABA | Completar la suscripción de la aplicación y guardar el canal con un token dedicado |
| **141006**, bloqueo de facturación | La cuenta tenía un problema de pagos para iniciar conversaciones | Incidencia independiente; no confundirla con la recepción de mensajes |

## 1. Error #3441034: lo que me desbloqueó

Comprobé permisos, empresa verificada, Embedded Signup, Facebook e Instagram. Nada resolvía el rechazo.

La pista apareció **en el dispositivo donde estaba WhatsApp Business**:

1. Abrí **WhatsApp Business → Ajustes → Cuenta → Plataforma para empresas** (*Business Platform*; los nombres pueden variar).
2. Vi **«Bandeja de entrada de Meta Business Suite»** como plataforma conectada.
3. Desconecté **esa integración**, no la cuenta de WhatsApp ni su número.
4. Volví a iniciar **«Seguir usando WhatsApp Business» / coexistencia** desde mi aplicación, y el **#3441034 dejó de bloquearme**.

Al pulsar «Desconectar» también recibí «Se produjo un error. Vuelve a intentarlo». Tras reiniciar el dispositivo lo conseguí. **No puedo demostrar que reiniciar fuera la causa**; pudo coincidir con un problema temporal.

**Precaución:** la desconexión retira la integración con Meta Business Suite Inbox y puede interrumpir su bandeja. No pulses a ciegas «Suprimir del porfolio empresarial» ni «Eliminar cuenta de WhatsApp»: son operaciones distintas y sus avisos pueden incluir consecuencias irreversibles.

Bird documenta `#3441034` como un rechazo de elegibilidad que puede requerir comprobar vinculaciones, permisos y el flujo elegido. WACO también describe el caso de una conexión previa con Meta Business Suite. Mi experiencia **confirma mi caso**, no que todos los `#3441034` tengan esa causa.

## 2. Error #2494064: no confundas coexistencia con migración

Al probar desde otra cuenta/flujo, Meta respondió que el número **ya estaba registrado en WhatsApp**.

Eso es compatible con que el teléfono esté funcionando en WhatsApp Business. Si quieres **mantener la aplicación móvil**, debes usar una configuración de **coexistencia**, no el alta estándar para un número nuevo ni una migración que retire la aplicación.

No «liberes» el número borrando su cuenta para resolver este mensaje sin entender qué camino de conexión estás utilizando.

## 3. Error #1690130 después de interrumpir Embedded Signup

Después de superar el bloqueo inicial, avancé hasta la pantalla de configuración del negocio y la zona horaria. Retrocedí para cambiarla y desconecté accidentalmente la plataforma desde WhatsApp Business mientras el alta seguía en curso.

Desde entonces la ventana de Meta mostraba **#1690130**, diciendo que el identificador del porfolio no era válido. Sin embargo, **el porfolio existía y estaba verificado**.

No he encontrado una explicación pública oficial que permita atribuir con certeza ese error a la interrupción, a la zona horaria ni a un supuesto conflicto por tener app y WABA bajo la misma empresa. **Esas causas no deben presentarse como confirmadas.** Tampoco pude resolverlo cerrando ventanas, recargando, cambiando de dispositivo o desconectando/reconectando repetidamente.

La diferencia importante fue dejar de intentar reiniciar el alta y mirar **qué existía realmente en Graph API**.

## 4. El diagnóstico de Graph API que cambió el enfoque

Con un token autorizado hice consultas **de solo lectura** a Meta y al estado de mi backend. El resultado fue:

| Elemento | Estado comprobado |
| --- | --- |
| Número en Graph | `CONNECTED` |
| Plataforma | `CLOUD_API` |
| Sigue en WhatsApp Business | `is_on_biz_app: true` |
| WABA | Existía y contenía el número |
| `GET /<WABA_ID>/subscribed_apps` | **Vacío** |
| Canales locales de mi aplicación | **Ninguno** |
| Webhook del número | Configurado hacia el backend |
| Campos del webhook | `messages`, `smb_message_echoes`, `history`, `smb_app_state_sync`, `account_update` ya configurados |

Es decir: **Meta parecía haber completado una parte importante de la coexistencia, pero mi software todavía no recibía los eventos de ese WABA ni tenía un canal persistido**.

Tres comprobaciones reutilizables para desarrolladores, con los identificadores sustituidos por marcadores:

```http
GET https://graph.facebook.com/vXX.X/<PHONE_NUMBER_ID>?fields=platform_type,is_on_biz_app
GET https://graph.facebook.com/vXX.X/<WABA_ID>/phone_numbers
GET https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
Authorization: Bearer <TOKEN_AUTORIZADO>
```

Los campos disponibles dependen de la versión de Graph, el tipo de cuenta y los permisos. Un error 403/100 **no demuestra por sí solo** que el número o la WABA no existan.

**Importante:** `CONNECTED` tampoco prueba que el usuario vea y reciba mensajes. En mi caso WhatsApp Manager mostraba «Fuera de internet», mientras Graph decía `CONNECTED`. La comprobación decisiva fue una prueba de mensaje real.

## 5. Recuperación sin volver a registrar el número

Como Graph indicaba `CLOUD_API` e `is_on_biz_app: true`, no quise ejecutar `/register`, borrar la WABA, dar de baja el número ni cambiar la propiedad de la aplicación.

Preparé un procedimiento **limitado a mi propia WABA y organización**, con comprobaciones previas y sin alterar otros clientes:

1. Creé un **usuario del sistema específico** con acceso solo a mi WABA y a la aplicación, con `whatsapp_business_management` y `whatsapp_business_messaging`.
2. Verifiqué el token y los permisos con consultas de lectura y comprobé que no podía consultar los WABA de otros clientes conocidos.
3. Guardé el token **solo en un almacén de secretos del servidor (Vault)**, no en el navegador ni en una tabla legible por usuarios.
4. Preparé y probé localmente la persistencia del canal y su aislamiento entre organizaciones.
5. Guardé el canal correspondiente al número **ya existente**.
6. Suscribí la app a esa WABA mediante `POST /<WABA_ID>/subscribed_apps`, tal como documenta la colección oficial de Meta.
7. Verifiqué por lectura que la suscripción era efectiva y el canal aparecía activo.

**Resultado observado el 2 de noviembre de 2026:** Meta devolvió **suscripción activa**, mi aplicación mostró el canal y **un mensaje nuevo enviado desde otro móvil llegó correctamente tanto a WhatsApp Business como a la bandeja de mi aplicación**.

Esto resolvió **la integración y recepción de mensajes** sin repetir Embedded Signup. **No prueba que el error #1690130 esté corregido dentro de la ventana de Meta**. Tampoco certifica por sí solo todos los envíos salientes, ecos, sincronización histórica o facturación.

Esta recuperación es una vía **avanzada para una integración propia** que ya tiene el número registrado y permisos adecuados. **No es un procedimiento general que cualquier usuario pueda ejecutar desde Meta Business Suite.** Antes de reproducirla, revisa la documentación vigente, los permisos, la propiedad del número, el aislamiento entre clientes, la idempotencia, los webhooks y el rollback.

### La operación de suscripción, con marcadores

```http
POST https://graph.facebook.com/vXX.X/<WABA_ID>/subscribed_apps
Authorization: Bearer <TOKEN_DE_SISTEMA_CON_PERMISOS>
```

No necesitas volver a configurar campos de webhook que ya están activos solo porque el WABA aún no tenga una app suscrita. Son **dos capas diferentes**.

## 6. ¿Por qué no aparecen conversaciones antiguas?

El nuevo canal recibe los eventos nuevos. Eso **no equivale a importar el historial del WhatsApp Business del móvil** ni a recuperar una copia de seguridad de un teléfono anterior.

La coexistencia puede ofrecer sincronización de contactos e historial con eventos `smb_app_state_sync` e `history`:

- El proveedor puede solicitarla mediante el endpoint `/<PHONE_NUMBER_ID>/smb_app_data`, si Meta todavía lo permite.
- La documentación de Gupshup advierte que cada sincronización es **de un solo uso** y debe solicitarse dentro de las **24 horas siguientes al onboarding**.
- La solicitud no garantiza recuperar chats que ya no estén presentes en WhatsApp Business; tampoco restaura automáticamente la historia local de una tablet.
- **En el momento de redactar este artículo no he confirmado una importación histórica satisfactoria de mi número.**

Por eso, antes de ejecutar una sincronización potencialmente irreversible, conviene verificar que **un mensaje entrante nuevo produce un webhook** y que el worker realmente guarda esos eventos.

## 7. Checklist práctico para investigar tu caso

1. **Identifica el error exacto** y en qué paso aparece: número, negocio, autorización, confirmación o envío.
2. **Si es #3441034**, mira primero WhatsApp Business → Ajustes → Cuenta → Plataforma para empresas. ¿Está conectada Meta Business Suite Inbox? Desvincula solo esa plataforma **si vas a sustituirla**.
3. **Si es #2494064**, verifica que usas **coexistencia** y no migración/registro estándar.
4. **Si es #1690130 tras un alta incompleta**, no des por hecho que el porfolio realmente es inválido. Inspecciona WABA, número y suscripción de la app mediante lecturas de Graph.
5. **Si Graph confirma coexistencia pero no hay webhooks**, comprueba `subscribed_apps`, callback y la persistencia local. No vuelvas a llamar a `/register` sin justificarlo.
6. **Haz una prueba real de mensaje entrante y saliente**, no te bases solo en `CONNECTED`.
7. **Si aparece 141006**, trata la facturación/envíos como otro incidente.
8. **Si no tienes acceso técnico suficiente**, abre un caso con Meta aportando código, fecha, identificador de sesión (por canal privado) y estados observados, sin publicar tokens ni números.

## Lo que **NO** afirmo

- No sé por qué Meta mostró exactamente **#1690130**: el origen interno del rechazo no está demostrado.
- No afirmo que cambiar la zona horaria causase el fallo.
- No afirmo que mover la app a otro porfolio sea la solución.
- No afirmo que cualquier `#3441034` se arregle desconectando Business Suite.
- No afirmo haber restaurado todos los chats antiguos o desbloqueado los mensajes iniciados por la empresa.

## Referencias

- [Meta for Developers — Onboarding WhatsApp Business app users (Coexistence)](https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users) (puede requerir sesión).
- [Meta / Postman — Subscribe to your WABA](https://www.postman.com/meta/whatsapp-business-platform/request/c1ai24q/subscribe-to-your-waba).
- [Bird — WhatsApp coexistence troubleshooting](https://bird.com/docs/guides/whatsapp/coexistence/troubleshooting).
- [WACO — My number is already in use on the WhatsApp Business app](https://docs.waco.id/en/existing-number).
- [Gupshup — Coexistence: history and contact sync window](https://partner-docs.gupshup.io/docs/co-existence-closed-beta-phase).

---

**Martín Muñoz** · [GitHub](https://github.com/martinmunozb) · Experiencia personal documentada el **2 de noviembre de 2026**.  

Si te ha ocurrido alguno de estos errores, deja el **código**, el **paso del flujo** y el **estado de `subscribed_apps`** (sin credenciales ni información personal) en un comentario o discusión. Eso ayuda a identificar qué soluciones funcionan realmente.
