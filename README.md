Cambios en el System_Prompt implementados:

**1. "D disponible" repetía la misma fecha en dos encabezados**

Dónde: sección ### BÚSQUEDA DE HORARIOS, bloque "Presentación — D disponible:", dentro del mismo párrafo que ya existía.

Texto exacto agregado:

"...nunca repetir la misma fecha en dos encabezados distintos, todas las horas de un mismo día van agrupadas bajo un único "📆 [fecha]:". 
Terminar de listar TODOS los días y horas que devolvió la herramienta antes de hacer la pregunta final."

**2. "D disponible" cortaba la lista antes de terminar (mostraba 4 días de 6)**

Dónde: "Presentación — D disponible:", inmediatamente a continuación del texto anterior en el mismo párrafo.

Texto exacto agregado:

"Antes de responder, contar mentalmente cuántos días distintos devolvió la herramienta (normalmente 6, salvo que la agenda esté más llena de lo habitual). 
La respuesta final debe tener exactamente esa misma cantidad de encabezados de fecha — ni uno menos. Si la herramienta devolvió 6 días y la respuesta solo menciona 4, 
es un error grave: significa que la lista se cortó antes de terminar. 
Nunca redactar la pregunta final ("¿cuál te queda mejor?") hasta haber escrito todos los días que devolvió la herramienta, uno por uno, sin excepción."

**El párrafo completo de "Presentación — D disponible" hoy, con 1. y 2. ya integrados:**

"Retorna todas las horas libres de los próximos días, sin límite. Mostrar TODAS, agrupadas por fecha, en orden cronológico, letras A, B, C… más allá de F si hace falta. 
Esto aplica sin importar cuántos resultados sean — 10, 20, 40 o más: nunca resumir, nunca cortar la lista a la mitad para preguntar antes de tiempo, nunca repetir la misma fecha en dos encabezados distintos,
todas las horas de un mismo día van agrupadas bajo un único "📆 [fecha]:". Terminar de listar TODOS los días y horas que devolvió la herramienta antes de hacer la pregunta final.
Antes de responder, contar mentalmente cuántos días distintos devolvió la herramienta (normalmente 6, salvo que la agenda esté más llena de lo habitual). 
La respuesta final debe tener exactamente esa misma cantidad de encabezados de fecha — ni uno menos. Si la herramienta devolvió 6 días y la respuesta solo menciona 4, 
es un error grave: significa que la lista se cortó antes de terminar.
Nunca redactar la pregunta final ("¿cuál te queda mejor?") hasta haber escrito todos los días que devolvió la herramienta, uno por uno, sin excepción."

**3. Problema: la variable nombre no se guardaba en la base de datos**

Esta fue la falla más persistente de la sesión — apareció tres veces, con matices distintos cada vez, hasta quedar resuelta de fondo.

Primer hallazgo — nombre completado en un mensaje aparte de la pregunta de precio.
Se probó: mensaje 1 = "Ana" (nombre de pila), mensaje 2 = "Torres, y aparte quisiera saber cuánto cuesta el blanqueamiento dental". El agente usó el nombre en el texto de la respuesta, pero nunca lo emitió como variable — el nodo que escribe en Clientes nunca se disparó.
Ubicación del arreglo: sección ### CAPTURA DE NOMBRE, párrafo nuevo después de "Si el primer mensaje menciona un servicio y/o un día...".
Texto agregado:


"Si el mensaje trae el nombre completo Y una pregunta de precio/servicio en el mismo mensaje (aunque no sea sobre un servicio puntual a agendar): llamar informacion Doc (única herramienta del ciclo) y además emitir la variable nombre en esa misma respuesta — no son dos acciones distintas, es una sola respuesta que combina el resultado de la herramienta con el dato que ya está disponible. Nunca omitir nombre solo porque en ese ciclo también se llamó una herramienta."
Validado: confirmado con una prueba real (nombre + precio combinados) — nombre se emitió correctamente.



El mismo bug reapareció con otra herramienta — Fecha Busqueda, no solo informacion Doc.
Con el número real del usuario, el mensaje "Jose Velasco, quiero limpieza para el martes" llamó a Fecha Busqueda y usó el nombre en el texto — otra vez sin emitir la variable. El arreglo anterior estaba atado específicamente a informacion Doc y no cubría esta combinación.
Ubicación del arreglo: mismo lugar, párrafo nuevo inmediatamente después del anterior.
Texto agregado:


"Esta regla aplica sin importar qué herramienta se llame en el ciclo (informacion Doc, Fecha Busqueda, H disponible, D disponible, Citas Disponibles): si el nombre completo está disponible en el mensaje actual y todavía no fue emitido, SIEMPRE incluir la variable nombre en la respuesta de ese mismo ciclo, junto con el resultado de la herramienta. Nunca usar el nombre solo en el texto de conversacion sin también emitirlo como variable — eso deja el dato sin guardar en la base, aunque la respuesta "suene" correcta."
Validado: confirmado con el número real del usuario — nombre y motivo se emitieron juntos, y el registro de la ejecución mostró "lastNodeExecuted":"nombre" (evidencia directa de que el nodo de guardado corrió, no solo que el JSON se veía bien).



Conexión con un diagnóstico anterior equivocado — "ventana de memoria".
Antes de encontrar la causa real de arriba, se había probado la persistencia a varios turnos (nombre → 2 mensajes de relleno → pedir horario) y el sistema pareció "olvidar" el servicio y la fecha, revirtiendo a datos viejos ("Diagnóstico"). En ese momento se atribuyó a contextWindowLength: 3 de Postgres Chat Memory. Ese diagnóstico fue incorrecto: al repetir la prueba con datos completamente limpios, se confirmó que el problema real era este mismo bug — el nombre nunca se guardaba desde el primer turno, por eso el sistema "revertía" a como si nunca lo hubiera tenido.

Validación final, con la causa raíz ya resuelta: se corrió la secuencia completa desde cero (nombre → 2 mensajes de relleno → pedir horario sin repetir nada) con el número real del usuario. El sistema recordó correctamente tanto el servicio como la fecha confirmada, sin volver a pedir el nombre. Cerrado.

Nota que sigue vigente: contextWindowLength: 3 en Postgres Chat Memory es una limitación real de la conversación (solo se recuerdan los últimos 3 mensajes) — no causó lo de arriba, pero podría ser relevante en otros escenarios no probados todavía.


**4. Problema: dos herramientas llamadas en el mismo ciclo**

Encontrado: el agente llamaba informacion Doc + D disponible (o Fecha Busqueda) juntas en la misma respuesta, violando "una sola herramienta por ciclo" — con mensajes muy densos (nombre + día + servicio + precio, todo junto).

Ubicación del arreglo: sección PASO 1 — CRITERIO ANTES DE ACTUAR, párrafo nuevo al final.
Texto agregado:


"¿El mensaje activa más de una necesidad a la vez (ej. una pregunta de precio junto con una fecha o servicio)? → elegir UNA sola acción para este ciclo — nunca llamar más de una herramienta, sin importar el tipo (informacion Doc, Fecha Busqueda, H disponible, D disponible, Citas Disponibles). Si hay una pregunta de información (precio/servicio) pendiente, esa gana el turno del ciclo; lo demás queda registrado para retomarlo en el ciclo siguiente."



Complemento — ubicación: sección VERIFICACIÓN ANTES DE ENVIAR, ítem 6 (nuevo):


"6. ¿Uso H disponible o D disponible para un día concreto? → ERROR CRÍTICO. Usar Fecha Busqueda."

**5.  Problema: Max iterations (3) reached en un mensaje que necesitaba 3 llamadas legítimas**

Encontrado: el agente necesitaba informacion Doc → Fecha Busqueda → Citas Disponibles (tres llamadas distintas, no redundantes, en iteraciones separadas) y se quedó sin margen antes de poder responder.

Por qué es distinto al problema de la sección 7: ahí el problema era llamar dos herramientas a la vez (mal). Acá el modelo ya estaba llamando una por vez, correctamente — el problema es que la tarea, de por sí, necesitaba más de 3 pasos totales.

Solución: no es un cambio de prompt — es el parámetro maxIterations del nodo AI Agent. Cambio: 3 → 4.

Validado: confirmado — no repite llamadas innecesarias (eso lo sigue evitando la regla de la sección 7), solo da un paso más de margen para tareas que de verdad lo requieren.
