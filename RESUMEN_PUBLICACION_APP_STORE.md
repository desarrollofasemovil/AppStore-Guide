# Cómo publicamos una app en la App Store — Resumen para nuevos

**Para quién es:** cualquier persona que llegue al área mobile de 101 SAS, sepa o no de programación.
**Tiempo de lectura:** 10 minutos.
**¿Necesitas el detalle técnico?** Está en la [guía completa](./GUIA_PUBLICACION_APP_STORE.md). Aquí cada tema dice a qué sección ir (§).

---

## La idea en una frase

Para que una app aparezca en la App Store, la empresa la **arma**, la **firma**, la **prueba** y se la **entrega a Apple para que la revise**. Como no tenemos computadores Mac, el armado lo hace un servicio en la nube llamado **Codemagic**.

---

## Las piezas, explicadas con una comparación

Piensa que publicar una app es como **sacar un producto a la venta en una App Store muy exigente**.

| Pieza | Qué es | En la comparación |
|---|---|---|
| **Cuenta Apple Developer** | La membresía anual de la empresa con Apple | El contrato de la empresa con el App Store. Si vence, no se puede vender nada |
| **Team "101 SAS"** | El grupo de personas de la empresa con acceso a esa cuenta | La lista de empleados autorizados a entrar a la bodega |
| **App Store Connect** | La página web de Apple donde se administran las apps | La oficina de la App Store: ahí se registra el producto, se pone la etiqueta y se pide la aprobación |
| **Codemagic** | Un Mac en internet que convierte el código en la app instalable | La fábrica que empaca el producto |
| **Certificados y perfiles** | Archivos que prueban que la app es nuestra | El sello de la empresa en el empaque. Sin él, Apple no la recibe |
| **TestFlight** | La app de Apple para probar antes de publicar | Las muestras gratis para el equipo antes de ponerlo en el estante |
| **Revisión de Apple** | Una persona de Apple prueba la app y la aprueba o la rechaza | El control de calidad de la App Store |

---

## El camino, en 6 pasos

```
 1. Preparar  →  2. Armar  →  3. Probar  →  4. Perfilar (Ficha App Store)  →  5. Revisión  →  6. Publicar    
```

1. **Preparar la app.** El desarrollador revisa que la app cumpla lo básico que Apple exige (ver "Lo que Apple siempre revisa" más abajo). → §2
2. **Armarla en Codemagic.** Se oprime un botón y, en unos 20 minutos, Codemagic arma la app, la firma y se la envía a Apple. → §7.1
3. **Probarla en TestFlight.** El equipo la instala en un iPhone real y la usa como lo haría un cliente. **Es obligatorio:** como no tenemos Mac, esta es la primera vez que la app corre en un iPhone. → §7.3
4. **Llenar la ficha de la tienda.** Nombre, descripción, capturas de pantalla, política de privacidad y una **cuenta de prueba** para el revisor de Apple. Las capturas se hacen en Figma con la plantilla *Platano Screenshots*, que ya trae el tamaño correcto de iPhone. → §8 y §9.1
5. **Enviar a revisión.** Apple la revisa, normalmente entre unas horas y un par de días. Si la rechaza, explica por qué, se corrige y se vuelve a enviar. → §9.2 y §9.3
6. **Publicar.** Cuando Apple la aprueba, la publicamos el día que el cliente decida. → §9.4

**Para las actualizaciones** se repiten los pasos 2 a 6. La preparación grande (paso 1 y la configuración de cuentas) se hace una sola vez por app.

---

## Lo que Apple siempre revisa (y nos ha costado rechazos)

No necesitas saber arreglarlos. Sí necesitas saber que existen, para pedirle al desarrollador que los revise **antes** de enviar:

| Lo que pasó | En palabras simples | Cómo se evita |
|---|---|---|
| **Icono rechazado** | El icono tenía las esquinas transparentes, y Apple exige un cuadrado sólido | Icono sin transparencias (el iPhone redondea las esquinas solo) |
| **"La app dice que funciona en iPad"** | Nuestras apps están hechas para iPhone y en posición vertical; Apple exige más cosas si dices que también sirve en iPad | Declarar la app **solo para iPhone** |
| **Pregunta de cifrado en cada envío** | Apple pregunta si la app usa cifrado especial. Si no se responde de antemano, cada versión se queda esperando | Dejar la respuesta escrita dentro de la app |
| **Faltaban textos de permisos** | Si la app *podría* usar la cámara, las fotos o Face ID, Apple exige un texto que explique para qué, aunque la app nunca lo pida | Escribir esos textos en español, claros y específicos |
| **El revisor no puede entrar** | Si la app tiene login, el revisor necesita un usuario de prueba | Dar siempre una **cuenta demo** que funcione |

La explicación técnica de cada uno está en la §10 de la guía completa.

---

## Quién hace qué

| Necesito… | Lo hace |
|---|---|
| Pagar o renovar la membresía, aceptar contratos con Apple | **Account Holder:** Mauricio Villa Sierra (Contactar al área de Infraestructura y contabilidad) |
| Que me den acceso al team 101 SAS | Un **Admin** del team |
| Armar la app y subirla | Un desarrollador del área mobile con acceso a Codemagic |
| Textos, capturas y datos de la ficha | Desarrollador + diseñador + cliente |
| Decidir el día de publicación | El cliente |

**Al llegar al área:** pide que te inviten a App Store Connect con el rol que te corresponda y **activa la verificación en dos pasos** en tu cuenta de Apple. Nunca uses la cuenta de otra persona. → §3.2

---

## Cuánto tarda cada cosa (aproximado)

| Paso | Tiempo |
|---|---|
| Armar la app en Codemagic | 15–25 minutos |
| Apple procesa la app y la pasa a TestFlight | 5–30 minutos |
| Revisión de Apple | De unas horas a un par de días |
| Primera publicación de una app nueva (todo el proceso) | Calcula **1 a 2 semanas**, por la ficha, las capturas y la posibilidad de un rechazo |

---

## Tres reglas que no se rompen

1. **Las claves y archivos secretos solo van en el gestor de contraseñas del equipo** ⟦nombre⟧. Nunca por chat, correo ni dentro del código.
2. **Nunca se envía a revisión una versión que nadie probó en un iPhone real.**
3. **Cada rechazo nuevo se anota** en el ticket de Jira de la app y en la §10 de la guía, para que no nos vuelva a pasar.

---

## Palabras que vas a oír

| Palabra | Significado |
|---|---|
| **Build** | Una versión armada de la app, lista para instalar |
| **Bundle ID** | El "número de cédula" de la app ante Apple. No cambia nunca |
| **Pipeline** | La receta automática que sigue Codemagic para armar la app |
| **TestFlight** | Donde probamos la app antes de publicarla |
| **Rechazo (rejection)** | Apple no aprobó la app y explica por qué |
| **Ficha (listing)** | Lo que el público ve en la tienda: nombre, fotos, descripción |

---

*Basado en la publicación de 101 Fintech y Parque Arví (sept. 2026). Última actualización: 2026-10-02. Responsable: Brayan Romero.*
