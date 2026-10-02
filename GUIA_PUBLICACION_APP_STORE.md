# Publicar una app Flutter en la App Store — Guía de punta a punta

**Equipo:** 101 SAS (1cero1 S.A.S.) · Área mobile
**App de referencia:** 101 Fintech — primera app publicada con este pipeline (sept. 2026), reutilizado después en Parque Arví
**Responsable del documento:** Brayan Romero · **Última actualización:** 2026-10-02
**Jira:** FSM — "Runbook de publicación iOS del área" (épica FSM-1)
**Versión en inglés:** [`IOS_APP_STORE_RELEASE_GUIDE.md`](./IOS_APP_STORE_RELEASE_GUIDE.md) (mismo contenido)

> **Por qué existe.** El área no tiene Mac y, hasta ahora, solo una persona sabía publicar en la App Store. Esta guía permite que cualquiera del equipo lleve una app Flutter del repo a "Ready for Sale" usando Codemagic, sin depender de esa persona.

> Los nombres de pantallas y botones de Apple y Codemagic se dejan **en inglés**, porque así aparecen en las consolas.

---

## Contenido

0. [Referencia rápida](#0-referencia-rápida)
1. [Panorama general](#1-panorama-general)
2. [¿La app está lista? Requisitos mínimos](#2-la-app-está-lista-requisitos-mínimos)
3. [Acceso: cuenta de Apple y team 101 SAS](#3-acceso-cuenta-de-apple-y-team-101-sas)
4. [Codemagic: API key, certificados y perfiles](#4-codemagic-api-key-certificados-y-perfiles)
5. [Configuración por app en Apple](#5-configuración-por-app-en-apple)
6. [El `codemagic.yaml` reutilizable](#6-el-codemagicyaml-reutilizable)
7. [Build y TestFlight](#7-build-y-testflight)
8. [Capturas de pantalla con Figma (Platano Screenshots)](#8-capturas-de-pantalla-con-figma-platano-screenshots)
9. [Ficha de la tienda y envío a revisión](#9-ficha-de-la-tienda-y-envío-a-revisión)
10. [Errores ya vistos (y cómo resolverlos)](#10-errores-ya-vistos-y-cómo-resolverlos)
11. [Checklist para una app nueva (copiar al ticket de Jira)](#11-checklist-para-una-app-nueva-copiar-al-ticket-de-jira)
12. [Inventario de secretos y bus factor](#12-inventario-de-secretos-y-bus-factor)
13. [Glosario](#13-glosario)

---

## 0. Referencia rápida

| Dato | Valor |
|---|---|
| Apple Developer Program | **Organización**, entidad legal "101 SAS" |
| Team ID | `43N22346WS` |
| Account Holder | Mauricio Villa Sierra — el único que paga la membresía y acepta acuerdos |
| Nombre de la integración en Codemagic | `101Software App Store` (se referencia tal cual en cada `codemagic.yaml`) |
| Máquina de Codemagic | `mac_mini_m2`, disparo manual, **tope de 30 min** |
| Tiempo típico de build | ~15–25 min |
| Xcode mínimo para subir | **Xcode 26 / SDK de iOS 26** (exigencia de Apple desde el 2026-04-28) → `xcode: latest` |
| Pipeline de referencia | `desarrollofasemovil/101Fintech` → `codemagic.yaml`, `docs/release/LANZAMIENTO_APP_STORE.md` |

**Apps que ya usan este pipeline**

| App | Bundle ID | Apple ID (numérico) |
|---|---|---|
| 101 Fintech | `com.x101Fintech.fintech101` | `6815744048` |
| Parque Arví | `com.tramites1cero1.parquearvi` | `6817031302` |

> El bundle ID de iOS **no tiene que coincidir** con el paquete de Android. 101 Fintech usa `com.x101Fintech.fintech101` en iOS y `com.x101Fintech.x101Fintech` en Android.

---

## 1. Panorama general

```
 UNA VEZ POR EQUIPO                     UNA VEZ POR APP                        CADA RELEASE
 ──────────────────                     ───────────────                        ────────────
 Membresía Apple Developer   ──┐        App ID (Identifiers) + capacidades     Subir versión en pubspec
 API key de App Store Connect ─┼──►     Ficha de la app en App Store Connect ► Lanzar build en Codemagic
 Integración en Codemagic    ──┤        Perfil de aprovisionamiento App Store  ↓
 Certificado Apple Distribution┘        codemagic.yaml en el repo              TestFlight (prueba interna)
 Clave APNs (si hay push)               App en Codemagic + grupo de secretos   ↓
                                        Ficha de la tienda + capturas          Enviar a revisión → Publicar
```

Qué se toca en cada lugar:

| Dónde | Qué pasa ahí |
|---|---|
| **developer.apple.com** (Certificates, Identifiers & Profiles) | App IDs, certificados, perfiles, claves APNs |
| **appstoreconnect.apple.com** | Usuarios, API keys, fichas de apps, TestFlight, ficha de tienda, revisión |
| **codemagic.io** | Compila el IPA en un Mac en la nube, lo firma y lo sube |
| **El repo** | `codemagic.yaml`, `Info.plist`, `project.pbxproj`, iconos, versión en `pubspec.yaml` |

---

## 2. ¿La app está lista? Requisitos mínimos

Antes de gastar minutos de macOS, la app Flutter tiene que cumplir lo básico. Cada punto de aquí sale de un rechazo real o de un build desperdiciado.

### 2.1 Configuración del proyecto (`ios/`)

| Requisito | Dónde | Por qué |
|---|---|---|
| Bundle ID único, puesto en todas las configuraciones | `ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER` | Debe coincidir exactamente con el App ID y el perfil |
| `DEVELOPMENT_TEAM = 43N22346WS` | `project.pbxproj` | Firma bajo el team 101 SAS |
| **Solo iPhone:** `TARGETED_DEVICE_FAMILY = 1` en las **tres** configuraciones de Runner (Debug/Release/Profile) | `project.pbxproj` | Evita el error 90474 y la exigencia de capturas de iPad — ver §10.2 |
| Deployment target que acepten tus plugins | `IPHONEOS_DEPLOYMENT_TARGET` | 101 Fintech usa **15.0** porque lo exige el Firebase iOS SDK 12 |
| `ITSAppUsesNonExemptEncryption = false` | `ios/Runner/Info.plist` | Si no, cada build queda en "Missing Compliance" — ver §10.3 |
| Un texto de propósito para **cada** API de privacidad que enlace un plugin | `Info.plist` (`NS…UsageDescription`) | Si no, ITMS-90683 — ver §10.4 |
| Iconos **sin canal alfa** (todos los tamaños, sobre todo el de 1024×1024) | `ios/Runner/Assets.xcassets/AppIcon.appiconset/` | Si no, ITMS-90717 — ver §10.1 |
| Capacidad de push en `Runner.entitlements` solo si la app de verdad usa push | `ios/Runner/Runner.entitlements` | El perfil debe incluirla o falla la firma — ver §10.6 |

> **¿Sin Mac? Edita los archivos como texto.** `project.pbxproj` e `Info.plist` son texto plano. Todos los arreglos de esta guía se hicieron sin abrir Xcode.

> **Dependencias:** con Flutter 3.44 las dependencias nativas de iOS van por **Swift Package Manager** por defecto, y 101 Fintech **no tiene Podfile**. Si una app vieja todavía usa CocoaPods, `flutter build ipa` corre `pod install` solo — no hay que agregar nada al pipeline.

### 2.2 Requisitos de producto (lo que revisa Apple)

- **Versión:** `pubspec.yaml` → `version: X.Y.Z+N`. `X.Y.Z` es lo que ve el usuario; `N` es el número de build (el pipeline lo sube solo si hace falta).
- **Cuenta demo** si hay cualquier login. El revisor no pasa de la pantalla de ingreso sin una, y eso es rechazo automático. 101 Fintech trae un modo demo en el mismo binario de release (documento `9000000101`, credenciales en `docs/release/CUENTA_DEMO_REVISION.md` de ese repo — **no** se copian aquí).
- **URL de política de privacidad** — obligatoria para toda app, aunque no recoja nada.
- **URL de soporte** — obligatoria.
- **Eliminación de cuenta** dentro de la app si la app permite *crear* cuentas (Guideline 5.1.1(v)).
- **Apps financieras** las debe publicar la organización, no una persona (Guideline 5.1.1(ix)) — 101 SAS ya cumple.
- **Usar los nombres de Apple para sus funciones:** en iPhone se dice "Face ID", no "huella". 101 Fintech tuvo que cambiar los textos del ingreso biométrico para iOS (2026-09-25).
- **Nada de callejones sin salida:** botones que no hacen nada, pantallas de "próximamente" o contenido de relleno son un rechazo común (Guideline 2.1, App Completeness).

---

## 3. Acceso: cuenta de Apple y team 101 SAS

### 3.1 Roles que importan

| Rol | Qué puede hacer | Quién debería tenerlo |
|---|---|---|
| **Account Holder** | Pagar/renovar la membresía, aceptar acuerdos legales, todo lo demás | Una sola persona (Mauricio Villa Sierra) |
| **Admin** | Invitar usuarios, crear API keys, gestionar certificados; todo menos acuerdos/pagos | **Al menos dos personas del área mobile** |
| **App Manager** | Gestionar apps, TestFlight, enviar a revisión | Desarrolladores que publican |
| **Developer** | Subir builds, TestFlight | Desarrolladores que solo compilan |

### 3.2 Meter a una persona nueva en el team

1. Un **Admin** (o el Account Holder) entra a **App Store Connect → Users and Access → "+"**.
2. Pone nombre y correo de la persona, elige el rol (normalmente **App Manager**) y marca **"Access to Certificates, Identifiers & Profiles"** si va a manejar firmas.
3. La persona acepta la invitación por correo, entra con su Apple Account (o crea una) y **activa la verificación en dos pasos**.
4. Verifica que ve el team **"101 SAS"** en el selector de arriba a la derecha, tanto en developer.apple.com como en App Store Connect. Si pertenece a varios teams, debe cambiar a 101 SAS antes de hacer cualquier cosa.

### 3.3 Lo que aprendimos al recuperar la cuenta (sept. 2026)

La cuenta se recuperó con una llamada a Apple (2026-09-21) y la membresía estaba **vencida desde el 2022-09-09**. Para no repetirlo:

- **Fecha de renovación** en un calendario compartido con recordatorio a 30 días. Una membresía vencida bloquea las subidas y puede sacar apps de la venta.
- **Acuerdos pendientes** (App Store Connect → Business) solo los acepta el Account Holder. Si una subida o un build de TestFlight queda bloqueado "por acuerdos", esa es la causa.
- **Mínimo dos Admins**, cada uno con su propia Apple Account y su propio teléfono de confianza. Nunca un login compartido.
- Guarda el número de caso de soporte de Apple y los datos de la membresía en el gestor de secretos (§12).

---

## 4. Codemagic: API key, certificados y perfiles

Esto es **a nivel de equipo** y se hace **una sola vez**. Una app nueva reutiliza todo, salvo el perfil de aprovisionamiento.

### 4.1 API key de App Store Connect (una vez por equipo)

1. **App Store Connect → Users and Access → Integrations → App Store Connect API → Team Keys → "+"**.
2. Nombre: `Codemagic`. Acceso: **App Manager**.
3. **Descarga el `.p8` de inmediato — Apple solo deja descargarlo una vez.** Anota el **Issuer ID** (arriba en la página) y el **Key ID**.
4. Guarda el `.p8`, el Issuer ID y el Key ID en el gestor de secretos (§12).

### 4.2 Integración de equipo en Codemagic (una vez por equipo)

1. **Codemagic → Team settings → Integrations → Developer Portal → Connect**.
2. Nombre: **`101Software App Store`** — exactamente ese, porque todos los `codemagic.yaml` lo referencian por nombre.
3. Pega el Issuer ID y el Key ID y sube el `.p8`.

> Si el nombre difiere en un solo carácter, el build falla al arrancar con "integration not found". Se corrige el nombre, no el yaml.

### 4.3 Certificado Apple Distribution (una vez por equipo)

Un solo certificado de distribución firma **todas** las apps del team. Apple permite pocos a la vez, así que no se crea uno por app.

1. **Codemagic → Team settings → codemagic.yaml settings → Code signing identities → iOS certificates → Generate certificate**.
2. Tipo **Apple Distribution**, usando la integración `101Software App Store`.
3. Codemagic muestra una contraseña para el `.p12`. **Guárdala en el gestor de secretos** y descarga también el `.p12`: Codemagic lo conserva, pero es la única otra copia.

Si ya existe (para 101 SAS ya existe), sáltate este paso — solo verifica que aparece en la lista y que no ha vencido (duran un año).

### 4.4 Perfil de aprovisionamiento (una vez por app)

1. **Code signing identities → iOS provisioning profiles → Fetch profiles**, y elige el perfil **App Store** del bundle ID de la app.
2. Si no existe, créalo en **developer.apple.com → Profiles → "+" → App Store Connect**, seleccionando el App ID y el certificado de distribución de arriba, y vuelve a buscarlo.

> **Un perfil es una foto de las capacidades del App ID en ese momento.** Si después agregas una capacidad (por ejemplo, Push Notifications), el perfil viejo no la incluye: regenéralo y vuelve a traerlo en Codemagic.

### 4.5 Agregar la app a Codemagic (una vez por app)

1. **Codemagic → Add application → GitHub → repo** (organización `desarrollofasemovil`), tipo de proyecto **Flutter**.
2. En la app, **"Switch to YAML configuration"**. Desde ahí Codemagic lee el `codemagic.yaml` de la rama que se compila; el editor visual deja de aplicar.
3. Si la app tiene secretos: **Environment variables → nuevo grupo** (p. ej. `api_miapp`), con cada variable marcada como **Secure**. El nombre del grupo debe coincidir con `environment.groups` del yaml.

---

## 5. Configuración por app en Apple

Se hace una vez por cada app nueva, antes del primer build.

1. **Registrar el App ID** — developer.apple.com → Identifiers → "+" → App IDs → App:
   - Descripción: nombre de la app. Bundle ID: **Explicit**, p. ej. `com.x101Fintech.fintech101`.
   - Marca las capacidades que la app use de verdad (**Push Notifications** si usa FCM, Sign in with Apple, Associated Domains…). Hazlo **antes** de crear el perfil (§4.4).
2. **Crear la ficha de la app** — App Store Connect → Apps → "+" → New App:
   - Plataforma iOS, nombre (≤30 caracteres, único en la tienda), idioma principal (español), el bundle ID del paso 1, SKU (interno, cualquier valor único, p. ej. `fintech101-ios`), User Access: Full.
3. **Anotar el Apple ID numérico** — App Information → General → *Apple ID* (p. ej. `6815744048`). Va en `APP_STORE_APPLE_ID` del yaml.
4. **Solo si hay push — clave APNs.** developer.apple.com → Keys → "+" → *Apple Push Notifications service (APNs)*. Descarga el `.p8` (una sola vez), anota el Key ID y súbelo en **Firebase Console → Configuración del proyecto → Cloud Messaging → configuración de app de Apple**. Una clave APNs sirve para **todas** las apps del team: créala una vez y reutilízala. Sin ella, el push de FCM nunca llega a los iPhone y **nada muestra un error**. *(Estado: pendiente para 101 Fintech — en FSM.)*

---

## 6. El `codemagic.yaml` reutilizable

La plantilla está en esta carpeta: **[`codemagic.template.yaml`](./codemagic.template.yaml)** (sus comentarios están en inglés). Cópiala a la raíz del repo de la app como `codemagic.yaml` y reemplaza cada `⟦CHANGE⟧`.

### 6.1 Qué cambiar en cada app

| Clave | Valor en 101 Fintech | Tu app |
|---|---|---|
| `environment.flutter` | `3.44.0` | La misma versión que usas en local / CI de Android |
| `environment.groups` | `api_101fintech` | Tu grupo de secretos, o quitarlo |
| `ios_signing.bundle_identifier` y `vars.BUNDLE_ID` | `com.x101Fintech.fintech101` | Tu bundle ID |
| `vars.APP_STORE_APPLE_ID` | `6815744048` | El Apple ID numérico de §5.3 |
| `vars.BUILD_NUMBER_MINIMO` | `16` | Empieza en `1`; súbelo solo tras un build quemado (§10.5) |
| Paso "Compose env.json" | 4 secretos vía `jq` | Bórralo si la app no tiene secretos `--dart-define` |
| `--dart-define-from-file=env.json` | presente | Solo si conservaste el paso anterior |

**No cambiar:** `integrations.app_store_connect`, `instance_type`, `max_build_duration`, `xcode: latest`, el paso de firma, el de número de build y `publishing`.

### 6.2 Por qué está hecho así

| Decisión | Razón |
|---|---|
| **Sin bloque `triggering`** (disparo manual) | Los minutos de macOS los comparten todas las apps del equipo. Un push rutinario no debe gastarlos; publicar es un acto deliberado. |
| **`max_build_duration: 30`** | Un build atascado (firma, subida colgada) podía comerse 60 minutos. Uno sano tarda 15–25. Se bajó en el PR #72. |
| **Solo iOS en Codemagic** | Android ya compila en GitHub Actions (Linux, barato). Duplicarlo aquí desperdicia tiempo de Mac. |
| **`xcode-project use-profiles`** | Pasa el proyecto Flutter a firma manual con el perfil descargado y escribe `export_options.plist`. Sin esto, `flutter build ipa` intenta la firma automática y falla en un Mac de CI. |
| **Número de build desde TestFlight + suelo** | Cada subida de la misma versión necesita un `CFBundleVersion` único. Preguntarle a TestFlight el último y sumar 1 permite relanzar un release sin tocar el pubspec. El suelo cubre el caso que TestFlight no ve (§10.5). |
| **`--obfuscate --split-debug-info` + artefactos de símbolos** | Los binarios de release van ofuscados; sin los símbolos de Dart y los dSYM, las trazas de Crashlytics quedan ilegibles **para siempre** — recompilar el mismo commit no los regenera. |
| **`submit_to_testflight: false`** | El build queda para testers **internos** sin revisión de Apple. Pasar a testers externos o a revisión es una decisión humana en App Store Connect. |
| **Chequeo de variables que falla rápido** | Un secreto faltante compilaría bien y fallaría en el primer login con un error que no apunta a la causa. Fallar en el pipeline nombra la clave que falta (nunca su valor). |
| **Paso "Pre-flight checks"** *(nuevo en la plantilla)* | Falla en un segundo si falta la clave de cifrado en el plist, si la app declara iPad o si el icono de 1024 tiene alfa — los tres errores que ya pagamos. ⚠️ Se agregó para la plantilla y **todavía no ha corrido en Codemagic**; si falla en la primera app que lo use, corrígelo o quítalo. |

---

## 7. Build y TestFlight

### 7.1 Lanzar un build

1. Sube `version:` en `pubspec.yaml` si es una versión nueva, haz commit y push.
2. **Codemagic → la app → Start new build → rama → workflow "iOS · TestFlight"**.
3. Sigue el log. En orden: Dependencies → (env.json) → Signing → Build number (imprime `Version: X.Y.Z (N)`) → Pre-flight → Build IPA → Package symbols → Publishing.
4. Al terminar, **descarga y archiva** `dart-symbols-ios-*.zip` y `dsyms-ios-*.zip` junto al release (drive del equipo, carpeta por versión). No es opcional — ver §6.2.

### 7.2 Qué pasa después de la subida

1. App Store Connect **procesa** el build (normalmente 5–30 min). Llega un correo cuando está listo — o un correo de rechazo con un código `ITMS-xxxxx` (ir a §10).
2. **TestFlight → iOS builds**: aparece el build. Gracias a `ITSAppUsesNonExemptEncryption`, no se queda en "Missing Compliance".
3. **Prueba interna:** TestFlight → Internal Testing → grupo → agregar el build. Los testers internos deben ser usuarios del team en App Store Connect (hasta 100). **Sin revisión de Apple.**
4. **Prueba externa** (cliente, gente fuera del equipo): requiere **Beta App Review** para el primer build de cada versión. Suele ser rápida, pero es una revisión.
5. Los testers instalan la app **TestFlight** en su iPhone y aceptan la invitación.

### 7.3 Prueba en un iPhone real (obligatoria)

Sin Mac, **el primer build de TestFlight es la primera vez que la app corre en iOS.** Antes de enviar a revisión, revisa como mínimo:

- [ ] Arranque en frío, splash y el flujo principal de punta a punta.
- [ ] Login con la **cuenta demo** (exactamente lo que usará el revisor).
- [ ] Cada solicitud de permiso muestra el texto correcto en español.
- [ ] Face ID / biometría (si hay), enlaces externos, PDF, redirecciones de pago.
- [ ] Push (si hay — necesita la clave APNs, §5.4).
- [ ] Áreas seguras (notch, Dynamic Island, indicador de inicio) en todas las pantallas.

---

## 8. Capturas de pantalla con Figma (Platano Screenshots)

### 8.1 Qué exige Apple (app solo iPhone)

| Pantalla | Tamaños aceptados (vertical, px) | ¿Obligatoria? |
|---|---|---|
| **6.9"** | **1320 × 2868**, 1290 × 2796, 1260 × 2736 | **Sí** — Apple las reduce para los iPhone más pequeños |
| 6.5" | 1284 × 2778, 1242 × 2688 | Solo como respaldo si no tienes de 6.9" |
| iPad 13" | 2064 × 2752 / 2048 × 2732 | **Solo si la app declara iPad** — otra razón para publicar solo iPhone |

- **De 1 a 10** capturas por tamaño de pantalla e idioma. Apunta a **5–8**.
- **PNG o JPEG, sin transparencia.** Un PNG puede llevar canal alfa aunque se vea opaco, y la subida se rechaza. Ante la duda, exporta en **JPG**.
- Opcional: hasta 3 vistas previas en video (app previews) — no las hemos usado.

### 8.2 Flujo con Platano Screenshots

La plantilla de Figma **Platano Screenshots** ya trae los marcos de iPhone en los tamaños que exige Apple. Link de la plantilla: ⟦[Link de Figma Community](https://www.figma.com/design/8W8KiYgIpN5emBGZhg2CgK/Platano-Screenshots--Community-?node-id=15-1891&p=f&t=C6uyNLJwgbBUy2o5-0)⟧.

1. **Duplica** la plantilla en el espacio de Figma del equipo 101 (un archivo por app, p. ej. `AppStore Screenshots — <App>`).
2. **Consigue las pantallas reales de la app**, en este orden de preferencia:
   - **Desde un iPhone real vía TestFlight**, con la cuenta demo. Una captura de un iPhone Pro Max/Plus ya mide 1290 × 2796 o 1320 × 2868 — sirve tal cual.
   - **Desde el Figma del diseñador** (ver §8.3).
   - Desde Android, solo como último recurso: la barra de estado y la navegación se ven de Android, y Apple puede rechazar capturas que no muestren la app como corre en iOS.
3. **Usa datos fijos.** El modo demo de 101 Fintech usa una semilla fija (`DEMO_DATOS_ALEATORIOS=false`), así todas las capturas muestran la misma cuenta. Nunca uses datos de clientes reales.
4. Pon cada pantalla en un **marco de 6.9"**, escribe el titular (corto, orientado al beneficio, en español) y mantén el texto dentro del área segura.
5. **Exporta** cada marco a **1x**, en **JPG** (o PNG sin transparencia), con nombres en orden: `01_inicio.jpg`, `02_movimientos.jpg`…
6. Guarda los exportados en el repo o en el drive del equipo en `store/ios/screenshots/<versión>/`, para que el siguiente release arranque de ahí.

### 8.3 Reutilizar las capturas del diseñador

Si el diseñador ya hizo capturas para la tienda (por ejemplo para Google Play), reutilízalas — pero **cambia el tamaño del marco, no estires la imagen**:

1. Revisa la proporción original. Los diseños de Android/Play suelen ser 9:16 (1080 × 1920); el iPhone de 6.9" es ~9:19,5. **No** tienen la misma forma.
2. En Figma, crea un marco de exactamente **1320 × 2868** (o 1290 × 2796), ponle el mismo color o degradado de fondo del diseño y coloca dentro la composición original **escalada proporcionalmente** (mantén Shift o usa la herramienta Scale `K`). Rellena el espacio vertical sobrante con fondo, o sube el titular y dale más espacio al mockup del teléfono.
3. Si el diseño usa un mockup de un teléfono Android, cámbialo por el marco de iPhone de Platano Screenshots.
4. Exporta como en el paso 5 de §8.2 y verifica el tamaño en píxeles de cada archivo antes de subirlo (el panel de exportación de Figma lo muestra; en Windows, Propiedades del archivo → Detalles).

---

## 9. Ficha de la tienda y envío a revisión

### 9.1 Llenar la ficha (una vez por app; revisarla en cada versión)

| Sección | Campos | Notas |
|---|---|---|
| **App Information** | Nombre (≤30), subtítulo (≤30), categoría (principal + secundaria opcional), **Content Rights** | Content Rights: declarar si la app muestra contenido de terceros (fotos, videos) y que se tienen los derechos. Es una declaración legal — confirmar con el cliente. |
| **Pricing and Availability** | Precio (gratis), países | Preguntar al cliente: todos los países o solo Colombia. |
| **App Privacy** | URL de política de privacidad, cuestionario de datos recolectados | Responder a partir de las dependencias **reales** del `pubspec.yaml` (Firebase Analytics, Crashlytics, cualquier SDK que recoja datos), no de memoria. "Data Not Collected" solo vale si ningún SDK recoge nada. Después, **publicar** la sección de privacidad. |
| **Age Rating** | Cuestionario | La mayoría de nuestras apps quedan en 4+. Responde "Unrestricted Web Access" = Sí solo si la app deja navegar a cualquier URL. |
| **Página de la versión** | Capturas, texto promocional (≤170), descripción (≤4000), palabras clave (≤100, separadas por comas), URL de soporte, URL de marketing (opcional), copyright (`2026 101 SAS` o el cliente), "What's New" (no aplica en la primera versión) | Prepara los textos en un `.md` en el repo (Parque Arví usó `docs/release/PROMPT_COWORK_APP_STORE.md`) y pégalos; revisa los límites de caracteres antes de pegar. |
| **App Review Information** | Nombre, teléfono y correo de contacto; **sign-in required → usuario y contraseña demo**; notas | Notas: explica lo que no sea obvio (p. ej. "los pagos abren la pasarela PSE del banco en el navegador; la cuenta demo la simula"). |
| **Version Release** | Manual / automático tras la aprobación / programado | Usa **Manually release** para que el cliente decida el momento del lanzamiento. |

### 9.2 Enviar

1. Página de la versión → **Build** → "+" → elige el build de TestFlight que probaste.
2. Verifica que la respuesta de export compliance salió del plist (no aparece la pregunta).
3. **Add for Review → Submit to App Review**. El estado pasa por *Waiting for Review → In Review → Pending Developer Release* (o *Rejected*).
4. La revisión suele tardar desde unas horas hasta un par de días.

### 9.3 Si Apple rechaza

1. Lee el mensaje en **App Review → el envío** (Resolution Center). Cita un número de guideline.
2. Si es **solo de metadatos** (textos, capturas, cuenta demo), corrígelo en App Store Connect y responde/reenvía — sin build nuevo.
3. Si requiere **código**, corrígelo, lanza un build nuevo en Codemagic (misma versión; el pipeline sube el número de build), selecciona el build nuevo y reenvía.
4. Si crees que el revisor se equivoca, responde en el hilo con datos concretos (pasos, credenciales demo, video). Existen las apelaciones, pero una respuesta clara suele resolverlo.
5. Registra el motivo del rechazo en el ticket FSM de la app y, si es nuevo, **agrégalo a la §10 de esta guía.**

### 9.4 Después de la aprobación

- *Pending Developer Release* → **Release This Version** cuando el cliente esté de acuerdo.
- Actualizaciones siguientes: subir `version` en `pubspec.yaml` → build → TestFlight → nueva versión en App Store Connect ("+ Version") → llenar "What's New" → enviar.

---

## 10. Errores ya vistos (y cómo resolverlos)

| # | Síntoma | Código | Causa | Solución |
|---|---|---|---|---|
| 10.1 | Subida rechazada, "invalid large app icon… alpha channel" | **ITMS-90717** | Iconos con transparencia | Aplanar los iconos sobre un fondo opaco |
| 10.2 | Subida rechazada, la multitarea de iPad exige todas las orientaciones | **90474** | La app declara iPad (`"1,2"`) pero es solo vertical | `TARGETED_DEVICE_FAMILY = 1` |
| 10.3 | Cada build queda en "Missing Compliance" | — | Falta la declaración de cifrado | `ITSAppUsesNonExemptEncryption = false` |
| 10.4 | Build rechazado al procesarlo, "missing purpose string" | **ITMS-90683** | Un plugin enlaza una API de privacidad sin su `NS…UsageDescription` | Agregar los textos de propósito |
| 10.5 | La siguiente subida dice "build number already used" aunque TestFlight no lo muestra | — | Un build rechazado quemó su número | Subir `BUILD_NUMBER_MINIMO` |
| 10.6 | La firma o la subida falla por `aps-environment` | — | Perfil creado antes de activar Push en el App ID | Activar Push, regenerar el perfil y volver a traerlo |
| 10.7 | El build falla al arrancar, integración no encontrada | — | El nombre de la integración no coincide | Debe ser `101Software App Store` |
| 10.8 | Subida rechazada por versión del SDK | — | Compilado con Xcode < 26 | Mantener `xcode: latest` |

### 10.1 Canal alfa en los iconos — ITMS-90717

**Qué pasó (101 Fintech, 2026-09-24):** los 21 PNG de `AppIcon.appiconset` tenían las esquinas redondeadas transparentes. App Store Connect rechaza la subida si el icono de 1024×1024 tiene transparencia.

**Arreglo usado:** rellenar las esquinas transparentes con el color de marca (`#14387F`, el mismo fondo del icono adaptativo de Android). A la vista no cambia nada, porque **iOS redondea las esquinas por su cuenta** — el icono debe ser un cuadrado completo.

**Cómo prevenirlo en una app nueva:**

```yaml
# pubspec.yaml — flutter_launcher_icons
flutter_launcher_icons:
  ios: true
  image_path: "assets/images/logo.png"
  remove_alpha_ios: true
  background_color_ios: "#14387F"   # color de marca detrás de las partes transparentes
```

Luego `dart run flutter_launcher_icons`.

**Revisar antes de compilar** (funciona en Windows/Linux, necesita Python + Pillow):

```bash
python -c "import glob; from PIL import Image; [print(f, Image.open(f).mode) for f in glob.glob('ios/Runner/Assets.xcassets/AppIcon.appiconset/*.png')]"
```

Todas las líneas deben decir `RGB`. `RGBA` o `LA` = canal alfa → hay que corregir. Para aplanarlos a mano:

```bash
python -c "import glob; from PIL import Image; [Image.alpha_composite(Image.new('RGBA', (im:=Image.open(f).convert('RGBA')).size, '#14387F'), im).convert('RGB').save(f) for f in glob.glob('ios/Runner/Assets.xcassets/AppIcon.appiconset/*.png')]"
```

### 10.2 iPad / multitarea — error 90474

**Qué pasó (Parque Arví y 101 Fintech, 2026-09-25):** la plantilla de Flutter trae `TARGETED_DEVICE_FAMILY = "1,2"` (iPhone + iPad). Una app que soporta iPad con multitarea debe soportar las cuatro orientaciones, y nuestras apps van fijas en vertical (`Info.plist` + `SystemChrome.setPreferredOrientations`). App Store Connect rechazó la subida.

**Arreglo usado:** declarar la app **solo para iPhone**. En `ios/Runner.xcodeproj/project.pbxproj`, reemplazar en las **tres** configuraciones de Runner (Debug, Release, Profile):

```diff
- TARGETED_DEVICE_FAMILY = "1,2";
+ TARGETED_DEVICE_FAMILY = 1;
```

Verificar: `grep -n TARGETED_DEVICE_FAMILY ios/Runner.xcodeproj/project.pbxproj` → solo líneas con `= 1;`.

**Efectos secundarios (todos buenos para nosotros):** no se exigen capturas de iPad, y la app igual se instala en iPad en modo de compatibilidad con iPhone.

**Solo si el cliente de verdad quiere iPad:** deja `"1,2"` y, o soportas las cuatro orientaciones, o agregas `UIRequiresFullScreen = true` al `Info.plist` (desactiva la multitarea). En ese caso también necesitas capturas de iPad y probar en iPad.

### 10.3 `ITSAppUsesNonExemptEncryption`

**Qué pasó (Parque Arví, detectado al llenar la ficha, 2026-09-29):** sin esta clave, App Store Connect hace la pregunta de export compliance en **cada build**, y el build no se puede probar ni enviar hasta que alguien la conteste.

**Arreglo:** agregar en `ios/Runner/Info.plist`, dentro del `<dict>` raíz:

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

**Cuándo `false` es correcto:** la app solo usa el cifrado que trae iOS o protocolos estándar — HTTPS/TLS, Keychain/almacenamiento seguro, Face ID. Eso cubre todas nuestras apps hoy, incluida 101 Fintech. **Si alguna app llegara a usar criptografía propia**, debe ir en `true` y el equipo tiene que revisar la documentación de export compliance. No copies el `false` a ciegas en una app así.

### 10.4 Textos de permisos — ITMS-90683

**Qué pasó (101 Fintech build 2.0.2 (15), 2026-09-25):** Apple rechazó el build al procesarlo porque le faltaban `NSPhotoLibraryUsageDescription` y `NSCameraUsageDescription`. **La app nunca pide fotos ni cámara** — solo adjunta PDF —, pero `file_picker` compila su parte de galería y cámara (DKImagePickerController) y, con Swift Package Manager, no hay forma de excluirla. Apple escanea el binario, no el comportamiento.

**Regla:** toda API de privacidad **enlazada** en el binario necesita su texto de propósito, la llame la app o no.

**Arreglo usado:**

```xml
<key>NSFaceIDUsageDescription</key>
<string>101 Fintech usa Face ID para autenticarte de forma segura.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>101 Fintech solo accede a tus fotos si eliges adjuntar un documento desde tu galería.</string>
<key>NSCameraUsageDescription</key>
<string>101 Fintech solo usa la cámara si eliges fotografiar un documento para adjuntarlo.</string>
```

**Mapa común plugin → clave** (revisa el `pubspec.yaml` de cada app):

| Plugin | Claves que suele exigir |
|---|---|
| `local_auth` | `NSFaceIDUsageDescription` |
| `file_picker`, `image_picker` | `NSPhotoLibraryUsageDescription`, `NSCameraUsageDescription` (image_picker puede pedir además `NSMicrophoneUsageDescription` para video) |
| `geolocator`, `location` | `NSLocationWhenInUseUsageDescription` (+ `…AlwaysAndWhenInUse…` solo para segundo plano) |
| `mobile_scanner`, `camera` | `NSCameraUsageDescription` |
| `contacts_service`, `flutter_contacts` | `NSContactsUsageDescription` |
| `speech_to_text` | `NSSpeechRecognitionUsageDescription`, `NSMicrophoneUsageDescription` |

**Cómo escribir los textos:**
- En **español**, concretos, nombrando la app y el *porqué*: "X usa la cámara para fotografiar documentos que adjuntas". Los textos genéricos ("Necesitamos acceso a la cámara") son un motivo de rechazo conocido bajo la Guideline 5.1.1.
- **No agregues claves "por si acaso".** Apple marcó `NSLocationWhenInUseUsageDescription` como no obligatoria para 101 Fintech, y `permission_handler` empieza a compilar código de ubicación cuando ve esa clave. Agrega solo lo que pida el correo de rechazo o la documentación del plugin.

### 10.5 Número de build quemado

**Qué pasó:** el build 2.0.2 (**15**) se rechazó al procesarlo (el ITMS-90683 de arriba). Apple da el número 15 por usado, pero **TestFlight no lista los builds rechazados**, así que la consulta del "último build" del pipeline no lo veía y habría reutilizado el 15.

**Arreglo:** `BUILD_NUMBER_MINIMO: 16` en `codemagic.yaml`. Cada vez que un build se rechace al procesarlo, pon el suelo en *(ese número + 1)*. Súbelo; nunca lo bajes.

### 10.6 Capacidad de push vs. perfil de aprovisionamiento

`Runner.entitlements` declara `aps-environment` (el valor `development` es correcto incluso para App Store; al exportar se reemplaza por `production` desde el perfil). Si el App ID no tenía activado **Push Notifications** cuando se creó el perfil, el perfil no lo incluye y la firma o la subida falla. Activa la capacidad en el App ID → regenera el perfil App Store → **Fetch profiles** en Codemagic → vuelve a compilar.

Si la app **no** usa push, quita la clave `aps-environment` / el archivo de entitlements en vez de activar la capacidad.

### 10.7 Integración no encontrada

El yaml dice `app_store_connect: 101Software App Store`. La integración de equipo en Codemagic debe tener **exactamente** ese nombre (mayúsculas y espacios incluidos).

### 10.8 Xcode / SDK viejo

Desde el **2026-04-28**, App Store Connect solo acepta builds hechos con **Xcode 26+ / SDK de iOS 26**. `xcode: latest` nos mantiene al día. Si una app fija un Xcode más viejo por algún motivo, la subida será rechazada.

### 10.9 Rechazos de revisión que conviene anticipar

Todavía no nos han pasado, pero conviene estar preparados:

- **2.1 App Completeness** — el revisor no puede entrar → siempre dar una cuenta demo que funcione.
- **5.1.1** — textos de permisos vagos, o creación de cuenta sin poder eliminarla desde la app.
- **4.2 Funcionalidad mínima** — apps que son solo un sitio web envuelto. Nuestras apps municipales con mucho WebView deben mostrar valor nativo.
- **Usar "huella" o lenguaje de Android en iOS** — decir Face ID y usar capturas con aspecto de iOS.

---

## 11. Checklist para una app nueva (copiar al ticket de Jira)

**Repo (Flutter)**
- [ ] Bundle ID definido y puesto en `project.pbxproj` (todas las configuraciones) — §2.1
- [ ] `DEVELOPMENT_TEAM = 43N22346WS`
- [ ] `TARGETED_DEVICE_FAMILY = 1` ×3 — §10.2
- [ ] Deployment target compatible con los plugins
- [ ] `ITSAppUsesNonExemptEncryption = false` (si aplica) — §10.3
- [ ] Textos de propósito para cada API de privacidad enlazada por plugins — §10.4
- [ ] Iconos RGB, sin alfa (`remove_alpha_ios: true`) — §10.1
- [ ] Entitlement de push solo si se usa push — §10.6
- [ ] Cuenta demo / modo demo funcionando en un build de release — §2.2
- [ ] `codemagic.yaml` desde la plantilla, con todos los `⟦CHANGE⟧` reemplazados — §6

**Apple**
- [ ] App ID registrado con las capacidades correctas — §5.1
- [ ] Ficha de la app creada; Apple ID numérico copiado al yaml — §5.2–5.3
- [ ] Clave APNs subida a Firebase (si hay push) — §5.4

**Codemagic**
- [ ] App agregada desde GitHub y cambiada a YAML — §4.5
- [ ] Perfil de aprovisionamiento App Store traído — §4.4
- [ ] Grupo de secretos creado (si hace falta) — §4.5

**Release**
- [ ] Build OK y símbolos archivados — §7.1
- [ ] Prueba en un iPhone real vía TestFlight — §7.3
- [ ] Capturas de 6.9" exportadas (JPG, sin alfa) — §8
- [ ] Ficha completa: textos, privacidad (publicada), clasificación por edad, precio, URLs, copyright, datos para la revisión — §9.1
- [ ] Enviada con "Manually release" — §9.2
- [ ] Publicada y registrada en FSM

---

## 12. Inventario de secretos y bus factor

Todo lo de abajo debe vivir en el **gestor de contraseñas del equipo** ⟦nombre/ubicación⟧, nunca en un repo, chat o correo. Al menos dos personas deben tener acceso.

| Secreto | Dónde se crea | Quién lo usa | Rotación / vencimiento |
|---|---|---|---|
| Apple Accounts de los Admins (+ teléfonos de 2FA) | Apple | Personas | — |
| API key de App Store Connect (`.p8`, Issuer ID, Key ID) | App Store Connect → Integrations | Integración de Codemagic | Revocar y recrear si se filtra; el `.p8` solo se descarga una vez |
| Certificado Apple Distribution (`.p12` + contraseña) | Codemagic / developer.apple.com | Firma de todas las apps | **Vence cada año** — recordatorio en calendario |
| Clave APNs (`.p8`, Key ID) | developer.apple.com → Keys | Firebase (todas las apps) | No vence; revocar si se filtra |
| Secretos por app (p. ej. grupo `api_101fintech`) | El proveedor / cliente | Grupo de variables en Codemagic | Por app |
| Credenciales de la cuenta demo / de revisión | Docs del repo de la app | App Review Information | Cambiarlas si se filtran |
| Fecha de renovación de la membresía | Apple | Account Holder | **Anual** — recordatorio en calendario |

**Escalamiento:** bloqueado por acuerdos o pagos → Account Holder (Mauricio Villa Sierra). Bloqueado por acceso a la cuenta → cualquier Admin ⟦nombres⟧. Pipeline / firma → responsable del área mobile ⟦nombre⟧.

---

## 13. Glosario

| Término | Significado |
|---|---|
| **Bundle ID** | Identificador en notación DNS inversa de la app iOS (`com.x101Fintech.fintech101`). Es permanente una vez publicada la app. |
| **App ID** | El bundle ID registrado en developer.apple.com, con sus capacidades (Push, etc.). |
| **Apple ID (numérico)** | El número que App Store Connect le asigna a la ficha de la app (`6815744048`). No es secreto. |
| **Team ID** | Identificador del team de desarrollo 101 SAS (`43N22346WS`). |
| **Certificado de distribución** | Identidad criptográfica que firma los builds de App Store de todo el team. |
| **Perfil de aprovisionamiento** | Une un App ID + un certificado + las capacidades. Uno por app. |
| **Clave `.p8`** | Archivo de clave privada que Apple deja descargar una sola vez (API key, clave APNs). |
| **IPA** | El paquete de la app iOS que se sube. |
| **dSYM / símbolos de Dart** | Archivos que traducen las trazas de error ofuscadas a código legible. |
| **TestFlight** | La distribución beta de Apple. Testers internos: sin revisión. Externos: Beta App Review. |
| **`CFBundleShortVersionString` / `CFBundleVersion`** | Versión visible para el usuario (`2.0.3`) / número de build (`16`). Flutter los llena desde `pubspec.yaml`. |

---

*Fuentes de los requisitos de Apple al 2026-10-02: [Apple — SDK minimum requirements](https://developer.apple.com/news/upcoming-requirements/), [Especificaciones de capturas de App Store Connect (resumen)](https://appdrift.co/guides/app-store-connect/screenshot-requirements). Todo lo propio del equipo sale del repo de 101 Fintech (`codemagic.yaml`, `docs/release/`, commits 7825598, 1b42424, 8850343, cf3c309, 8b4414d) y del trabajo de publicación de Parque Arví.*
