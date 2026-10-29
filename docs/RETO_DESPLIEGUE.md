# Reto: despliega la Biblioteca del aula en AWS

Un equipo de desarrollo te entrega esta aplicación web. Funciona en su ordenador con `python app.py`. Tu trabajo es **dockerizarla y desplegarla en AWS**: la aplicación en un contenedor dentro de una **EC2** y los datos en una base de datos **PostgreSQL de Amazon RDS**.

Esta guía no tiene soluciones: tiene **preguntas**. Si sabes responderlas, sabrás qué hacer. Cuando no sepas, busca en la documentación oficial (Docker Hub, docs.docker.com, docs de Flask y de AWS) o vuelve a lo que hiciste con el Muro de notas.

---

## Antes de empezar

**Dónde trabajas:** en los PC del aula no hay Docker, así que todo lo de Docker se ejecuta en **tu EC2 del laboratorio**.
- El código y los ficheros Docker los editas en el IDE de tu PC y los llevas a la EC2 con `git push` (en el PC) y `git pull` (en la EC2), como en el Muro.
- Para ver la app en el navegador, usa un túnel SSH hacia el puerto que publiques en la EC2.

**Normas:**
- Trabaja en tu repositorio (creado desde la plantilla) y haz un commit al terminar cada fase.
- El fichero `.env` **nunca** se sube a GitHub. Compruébalo con `git status` antes de cada commit.
- No hace falta cambiar la lógica de la aplicación. Si crees que necesitas tocar `app.py`, piénsalo dos veces: casi siempre hay otra forma.

**Cuando algo falle**, antes de preguntar, recorre estas cuatro preguntas en orden:
1. ¿La imagen **se construye**? (mira la salida de `build`)
2. ¿El contenedor **sigue en marcha**? (mira `ps`: ¿`Up` o `Exited`?)
3. ¿Qué dicen los **logs**? (`logs` del contenedor que falla)
4. Si es la base de datos: ¿**no llega** (timeout: red o grupo de seguridad) o **llega y la rechaza** (contraseña, base de datos que no existe)?

---

## Fase 1 · Conoce la aplicación

Lee el `README.md` y `app.py` antes de escribir nada de Docker. No puedes desplegar algo que no sabes cómo funciona.

| Pregunta | Respuesta |
|---|---|
| ¿Qué versión de Python necesita? | |
| ¿Dónde están sus dependencias? | |
| ¿Qué variables de entorno lee? ¿Cuáles son obligatorias? | |
| ¿En qué puerto escucha? ¿En qué **dirección**? | |
| ¿Qué base de datos usa? ¿Quién crea la tabla, y cuándo? | |
| ¿Qué ficheros de la carpeta **no** hacen falta para ejecutarla? | |

<details><summary>Pista</summary>

La dirección en la que escucha no aparece en el README. Busca la línea `app.run(...)` y consulta en la documentación de Flask qué valor usa si no se indica `host`.
</details>

**Hecho cuando** sabes responder a todo y explicar qué pasa si falta `DATABASE_URL`.

---

## Fase 2 · Prepara AWS: la EC2 y la base de datos

Antes de dockerizar nada, deja preparadas las dos piezas de AWS y comprueba que se ven entre ellas.

1. ¿Qué recursos de AWS necesitas? Dibuja el camino: tu navegador, la EC2, el contenedor de la app y la RDS, con el **puerto** de cada flecha.
2. ¿Qué tiene que tener instalado la EC2? ¿Hace falta instalar Python o PostgreSQL en ella? ¿Por qué?
3. ¿Qué regla de entrada necesita el grupo de seguridad de la RDS? ¿Desde dónde? ¿Por qué no desde `0.0.0.0/0`?
4. ¿Debe tener la RDS **acceso público**? ¿Quién necesita llegar a ella?
5. La aplicación necesita una base de datos llamada `biblioteca`. ¿Existe al crear la RDS? Si no, ¿cómo la creas?
6. Escribe la `DATABASE_URL` completa para tu RDS. ¿Qué es cada parte?
7. ¿Cómo compruebas, desde la EC2 y **antes** de arrancar la app, que llegas a la RDS y que la contraseña es correcta? (No hace falta instalar nada: piensa en qué imagen de Docker trae un cliente de PostgreSQL.)

<details><summary>Pista para la pregunta 7</summary>

Un contenedor puede servir para ejecutar un solo comando y desaparecer (`docker run --rm ...`). La imagen oficial de PostgreSQL trae `psql`.
</details>

**Hecho cuando:**
- La EC2 ejecuta `docker run --rm hello-world` sin `sudo`.
- Desde la EC2 consigues conectarte a la base de datos `biblioteca` de tu RDS y ejecutar `SELECT 1;`.
- Sabes explicar qué significaría un *timeout* y qué significaría *password authentication failed*.

---

## Fase 3 · La imagen de la aplicación

Escribe el `Dockerfile` y el `.dockerignore`.

1. ¿Qué imagen base eliges? Compara `python:3.12`, `python:3.12-slim` y `python:3.12-alpine`: ¿qué cambia? ¿Cuánto ocupa tu imagen final?
2. ¿En qué orden copias los ficheros para que, si solo cambias `app.py`, Docker **no** vuelva a instalar las dependencias?
3. ¿Qué ficheros y carpetas de tu proyecto **no** deben entrar en la imagen? ¿Por qué es especialmente grave que entre `.env`?
4. ¿Con qué usuario se ejecuta la app dentro del contenedor? ¿Te parece buena idea que sea `root`?
5. ¿Qué comando arranca la aplicación cuando se inicia el contenedor?

<details><summary>Pista</summary>

Recuerda las capas del Dockerfile del Muro: cada instrucción es una capa, y Docker reutiliza las capas hasta la primera que cambia.
</details>

**Hecho cuando** `docker build` termina sin errores y has comprobado (entrando en un contenedor de esa imagen) que dentro no están `.env`, `.venv` ni `.git`.

---

## Fase 4 · Que responda

Arranca un contenedor con tu imagen, publicando el puerto y dándole la `DATABASE_URL` de tu RDS. Lo más probable es que **no funcione a la primera**. Esta es la fase más importante: no la saltes con una solución copiada.

1. ¿Cómo le das la `DATABASE_URL` al contenedor **sin** meterla en la imagen?
2. El contenedor está en marcha, pero no responde. ¿En qué dirección está escuchando la app **dentro** del contenedor? ¿Por qué eso impide que llegues desde fuera?
3. Mira los logs al arrancar. ¿Qué advertencia da Flask? ¿Qué significa?
4. ¿Qué es un servidor WSGI de producción? Elige uno. ¿Dónde lo declaras como dependencia?
5. La app tiene `debug=True`. ¿Por qué es peligroso en producción? ¿Sigue activo con tu solución?

<details><summary>Pista para la pregunta 2</summary>

Un servidor que escucha en `127.0.0.1` solo acepta conexiones que nacen en su misma máquina. ¿Cuál es "su misma máquina" cuando está dentro de un contenedor?
</details>

<details><summary>Pista para la pregunta 4</summary>

Un servidor de producción puede indicar él mismo en qué dirección y puerto escuchar. Quizá así no necesites tocar `app.py`.
</details>

**Hecho cuando** ves la lista de libros en el navegador (por el túnel), añades un libro, y el pie de página dice "Servido por:" seguido de algo que **no** es el nombre de la EC2. ¿Qué es ese nombre?

---

## Fase 5 · Compose: el despliegue descrito en un fichero

Arrancar el contenedor con un `docker run` larguísimo no es un despliegue que otro pueda repetir. Escribe un `docker-compose.yml`.

1. Solo tienes un contenedor. ¿Qué ganas usando Compose en lugar de `docker run`?
2. ¿Por qué en tu compose **no** hay un servicio de PostgreSQL?
3. ¿Dónde pones la `DATABASE_URL` y la `SECRET_KEY` para que no queden escritas en el `docker-compose.yml` ni en GitHub? ¿Qué pones en el repositorio en su lugar para que otro sepa qué variables necesita?
4. ¿Qué pasa si se te olvida definir una variable obligatoria? ¿Puedes hacer que Compose avise **antes** de arrancar?
5. ¿En qué dirección publicas el puerto en la EC2? ¿Qué cambia entre publicarlo en todas las interfaces o solo en `127.0.0.1`?
6. Si la EC2 se reinicia, ¿vuelve a arrancar tu contenedor solo? ¿Qué lo controla?

**Hecho cuando:**
- Un solo comando levanta la app y funciona en el navegador.
- En el `docker-compose.yml` no aparece ninguna contraseña.
- Si borras `DATABASE_URL` del `.env`, Compose se niega a arrancar y te dice qué falta.

---

## Fase 6 · ¿Dónde están los datos?

1. Añade libros, borra el contenedor (`down`) y vuelve a levantarlo. ¿Siguen los libros? ¿Por qué?
2. ¿Necesitas un volumen de Docker en este despliegue? ¿Por qué sí o por qué no?
3. ¿Qué pasa con los datos si **borras la EC2** entera? ¿Y si se para la RDS?
4. ¿Qué te da RDS que no te daría un contenedor de PostgreSQL dentro de la EC2? Piensa en copias de seguridad, actualizaciones y qué pasa si la EC2 se estropea.
5. Comprueba los datos directamente en la RDS, sin pasar por la web.

**Hecho cuando** puedes demostrar que los libros están en la RDS y no en el contenedor.

---

## Fase 7 · Actualizar

Una app desplegada no está terminada: cambia.

Haz un cambio visible en la app (por ejemplo, el título de la página o un texto del pie) y llévalo hasta AWS.

1. ¿Cuál es la secuencia completa, desde que guardas el fichero en el IDE hasta que el cambio se ve en el navegador?
2. ¿Qué comando de Docker usas en la EC2? ¿Qué pasaría si olvidas reconstruir?
3. Después de actualizar, ¿el contenedor es el mismo o uno nuevo? ¿Cómo lo compruebas?
4. ¿Qué cambia en el pie "Servido por:"? ¿Por qué?
5. Si cambias solo la `SECRET_KEY` en el `.env`, ¿hace falta reconstruir la imagen? ¿Por qué?

**Hecho cuando** el cambio se ve en AWS y los libros siguen ahí.
