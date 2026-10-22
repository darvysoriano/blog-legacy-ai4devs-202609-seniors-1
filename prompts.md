# Prompts

Aquí van **todos los prompts que lanzaste** para hacer el ejercicio, en el orden en que los
lanzaste, con el modelo y la herramienta de cada uno.

Esto no es papeleo. Lo que se revisa es **cómo pediste las cosas**, no solo lo que salió: un
resultado flojo con un prompt bueno y un resultado flojo con un prompt vago necesitan feedback
distinto, y sin este archivo no se distinguen.

## Cómo rellenarlo

- Un apartado `## Prompt N` por cada prompt.
- **Pega el prompt tal cual lo lanzaste**, dentro del bloque de código, aunque ocupe diez líneas
  y aunque tenga faltas. No lo reescribas para que quede bien: el que arreglaste mentalmente
  después no es el que lanzaste.
- Incluye también los que **no funcionaron**. Suelen ser los más útiles de leer.
- `Modelo` y `Herramienta` en todos. Si cambiaste de una a otra a mitad, se nota aquí.

Borra el ejemplo de abajo cuando escribas el primero.

---

## Prompt 1

**Modelo:** Opus 1M xHigh
**Herramienta:** Claude Code

```
realiza una auditoria de del código enlistando los errores encontrados, no resuelvas, sólo valida y enlista
```

**Qué salió:** 
  
  Auditoría del blog «Corriente»
Encontré 50 problemas. Lo revisé leyendo el código y los datos de la semilla. No pude probarlo en ejecución: npm test falla porque las dependencias del backend no están instaladas (Cannot find package '@poppinss/ts-exec'). Las cifras de recuentos que doy salen de calcular a mano con la semilla. No cambié nada.

🔴 Crítico: seguridad y datos personales
Se filtran correos en casi todas las respuestas públicas. Todas las consultas usan select * y devuelven la fila tal cual:
GET /posts devuelve posts.autor_email en cada post (routes.ts:74-87).
GET /posts/recientes devuelve autor_email (routes.ts:98).
GET /posts/:slug devuelve post.autor_email, autor.email y autor_email de cada comentario (routes.ts:166-174). Esto incumple la regla 6.
GET /posts/:slug/relacionados devuelve autor_email del post y autor.email (routes.ts:229-230).
GET /autores/:id devuelve autor.email y autor_email de cada post (routes.ts:289).
POST y PATCH de comentarios devuelven la fila con autor_email (routes.ts:333, routes.ts:392).
También salen campos internos: created_at, updated_at, estado e ids ajenos.
La clave de moderación está escrita en el código ('dev-secret-cambiar') y se reutiliza como "secreto de sesión" (routes.ts:25). La comparación tampoco es de tiempo constante (routes.ts:351).
Para borrar o editar un comentario basta con saber el correo del autor (routes.ts:349-357, routes.ts:378-381). Como esos correos se filtran (punto 1), cualquiera puede borrar cualquier comentario. Además, request.input también lee el correo de la query string, así que puede acabar en los logs de las URLs.
GET /autores/:id devuelve también los borradores, con el cuerpo entero (routes.ts:284-287). Incumple la regla 1. Los posts 9 y 10 son borradores con notas internas.
XSS guardado. El cuerpo del post se inyecta con dangerouslySetInnerHTML sin sanear (DetallePost.jsx:92). El post 1 de la semilla ya trae <img onerror=…>. Además, aHtml permite inyectar atributos a través del alt y del src de las imágenes en Markdown (DetallePost.jsx:18).
CSP y CSRF están desactivados (shield.ts). CORS tiene origin: true y credentials: true en desarrollo (cors.ts:21, cors.ts:42).
El POST de comentarios puede devolver el comentario de otra persona. Usa select * from comentarios order by id desc limit 1 (routes.ts:329-331): si llegan dos peticiones a la vez, puede devolver el comentario ajeno, con su correo.
🟠 Alto: reglas de negocio incumplidas
El recuento de comentarios no cuenta solo los aprobados (regla 5). Usa estado <> 'pendiente' (routes.ts:75, routes.ts:223), así que cuenta 'Aprobado', 'ok' y cualquier otro estado, por ejemplo uno de rechazo o spam.
El listado y el detalle dan cifras distintas. El detalle filtra con = 'aprobado' exacto (routes.ts:153). Con la semilla:
Post 1: el listado dice 4 comentarios y el detalle muestra 2.
Post 6: el listado dice 1 y el detalle muestra 0.
slug no es único en el esquema (posts migration:20). Incumple la regla 3: si hay slugs duplicados, el detalle devuelve uno cualquiera.
El filtro por categoría usa el nombre copiado dentro del post y no el id (routes.ts:54):
GET /categorias cuenta 2 posts en «Consejos», los posts 3 y 7 (routes.ts:242).
?categoria=Consejos devuelve solo 1, porque el post 7 todavía dice «Consejos de diseño».
El filtro y el recuento por etiqueta usan LIKE '%x%' sobre una cadena separada por comas (routes.ts:59, routes.ts:258):
Hay falsos positivos cuando una etiqueta está contenida en otra.
% y _ no se escapan.
En SQLite, LIKE no distingue mayúsculas en ASCII pero sí en «ñ» y en las vocales con tilde.
Las etiquetas se comparan con criterios distintos según el endpoint. El detalle compara el nombre exacto y distingue mayúsculas (routes.ts:145). Relacionados pasa todo a minúsculas (routes.ts:197). Si una etiqueta del candidato aparece repetida, puntúa dos veces (routes.ts:208-210).
GET /posts/recientes?limite=-1 devuelve todos los publicados (routes.ts:94-96). En SQLite, LIMIT -1 significa "sin límite". El código no comprueba que el valor sea mínimo 1 ni que sea entero.
GET /posts?pagina=1.5 pasa la validación y produce un offset decimal (routes.ts:39-47).
padre_id no se valida (routes.ts:308). Puede apuntar a un comentario que no existe o que es de otro post, y anidar a cualquier profundidad. El front solo pinta un nivel, así que las respuestas a respuestas desaparecen.
Al borrar un comentario padre, sus respuestas quedan huérfanas (routes.ts:359).
Anterior y siguiente no funcionan bien en los bordes. Si publicado_en es nulo, o si dos posts tienen la misma fecha, la navegación se rompe o salta posts (routes.ts:157-164).
🟠 Alto: validación y errores incoherentes
POST /posts/:slug/comentarios responde de cuatro maneras distintas según el fallo (routes.ts:310-321):
Falta el nombre: 400 {error}.
Falta el correo: 422 {message}.
El correo no tiene @: 400 {errores:[]}.
El cuerpo es corto: 200 {ok:false, motivo}.
La validación del correo solo busca una @. Un nombre hecho solo de espacios se acepta. No hay longitud máxima en ningún campo (routes.ts:316).
Se busca el post antes de validar la entrada (routes.ts:296-321).
La regla de autorización está duplicada y sale distinta en DELETE y en PATCH (routes.ts:352-353 frente a routes.ts:379):
DELETE normaliza el correo (minúsculas y sin espacios); PATCH lo compara tal cual.
PATCH no acepta la clave de moderación y exige que el comentario siga pendiente.
PATCH no exige mínimo de 5 caracteres en el cuerpo: se puede dejar en "a".
En PATCH se autoriza antes de validar el cuerpo (routes.ts:379-386).
Los errores no pasan por un manejador central. Cada ruta improvisa el suyo y handler.ts está sin tocar.
Las respuestas no tienen una forma común. Unas usan datos y otras post o autor; se mezclan porPagina (camelCase) con num_comentarios (snake_case).
🟡 Medio: arquitectura (frente a CLAUDE.md)
Toda la lógica del blog vive en routes.ts. No hay controladores, validadores VineJS, servicios, repositorios, modelos Lucid ni transformers del blog.
Hay db.rawQuery en las rutas y tipado any por todas partes.
Falta el alias #repositories/* en package.json.
El parseo de redes está duplicado (routes.ts:123-129, routes.ts:276-282). La consulta "post publicado por slug" se repite 3 veces.
🟡 Medio: esquema y datos
Hay datos copiados dentro de posts que ya no coinciden con la tabla original: autor_nombre, autor_email y categoria_nombre. En la semilla, el post 3 dice «Marta Ruiz», el post 5 tiene n.salgado@, y el post 7 dice «Consejos de diseño».
posts.etiquetas es una cadena y no hay tabla pivote.
comentarios.estado es texto libre, sin enum ni CHECK. En la semilla conviven 'aprobado', 'Aprobado' y 'ok'.
No hay claves foráneas en autor_id, categoria_id, post_id ni padre_id.
No hay índices. El slug de categorias tampoco es único.
creado_en se guarda como texto sin zona horaria (routes.ts:323).
🟡 Medio: rendimiento
GET /etiquetas hace N+1 consultas: una por etiqueta, 12 en total (routes.ts:256-262).
GET /relacionados carga todos los posts publicados en memoria y después hace 2 consultas por cada resultado (routes.ts:190, routes.ts:220-232).
El detalle hace 6 consultas en serie, y el front pide el mismo detalle dos veces: una en DetallePost y otra en Comentarios.
🟡 Medio: frontend
Los enlaces #/?etiqueta=… no filtran nada. ListaPosts solo lee categoria del hash (ListaPosts.jsx:15).
Hay dos contadores de comentarios que no coinciden. Uno está en la cabecera y otro en la sección de comentarios. El de la sección cuenta las respuestas y suma 1 al enviar un comentario que todavía está pendiente (Comentarios.jsx:39). Además, setTotal(total + 1) usa un valor antiguo del estado.
El detalle muestra el nombre de autor copiado y desactualizado (post.autor_nombre) junto al avatar de la tabla autores (DetallePost.jsx:68).
La URL de la API está copiada en 6 archivos, por ejemplo App.jsx:7.
ListaPosts fija 5 por página a mano en lugar de usar porPagina (ListaPosts.jsx:36). Cambiar de categoría lanza dos peticiones.
No se comprueba r.ok y los errores se tragan en silencio (.catch(() => {})). Tampoco se cancelan las peticiones viejas al cambiar de slug, así que una respuesta antigua puede pisar a la nueva.
Se usa el índice como key en todas las listas.
Los botones de compartir son href="#": no comparten nada y llevan a la portada (DetallePost.jsx:77-88).
La búsqueda enseña solo 5 resultados sin avisar de que hay más. El formulario de comentarios no tiene <label> y el campo de correo no es type="email".
⚪ Pruebas, herramientas y configuración
Solo hay un test, y comprueba un único campo (comentarios.spec.ts:41-42). Ninguno de los 9 endpoints del blog tiene cubierta la matriz mínima. El umbral de cobertura está en 0.
El hook de commits se puede saltar de dos formas (guard-tests.mjs:42, settings.json:5):
Solo intercepta la herramienta Bash. Un git commit lanzado desde la herramienta PowerShell pasa sin control.
La expresión \bgit\s+commit\b no detecta git -C . commit ni git -c x=y commit.
Quedan varios fallos de configuración:
.env.example:18 cita el puerto 5173, que está prohibido, y CORS_ORIGIN no lo lee nadie.
El Makefile usa rm, cp y test, que no funcionan en Windows sin una shell POSIX.
El entorno no tiene node_modules instalado, así que hoy la suite no arranca y el hook bloquearía cualquier commit.
