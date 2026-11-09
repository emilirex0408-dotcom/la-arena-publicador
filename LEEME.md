# Publicador de Instagram (GitHub Actions)

Publica el Reel a la hora aunque la laptop esté apagada. La API de Instagram no programa Reels, pero un contenedor
ya subido espera 24 h sin verse: la laptop lo sube y GitHub lo publica a la hora.

- Repo del workflow: `emilirex0408-dotcom/la-arena-publicador` (PRIVADO). Secreto `IG_TOKEN`.
- Repo de paso para videos largos: `emilirex0408-dotcom/la-arena-staging` (PÚBLICO, vacío casi siempre).

## Qué corre dónde

| Dónde | Qué hace |
|---|---|
| Laptop (`instagram_subir.py programar-todo`) | Mide el video. Crea el contenedor: rupload si dura menos de 55 s; si dura más, lo empuja a `la-arena-staging`, Instagram lo baja por `video_url` y la rama se vacía con force-push. Espera FINISHED y revisa `video_status`. Dispara el workflow con contenedor + hora. |
| Nube (`publicar-ig.yml`) | Espera dentro del runner hasta la hora (máx 5 h 40 min), lee `IG_TOKEN`, revisa el contenedor (status_code y video_status) y llama `media_publish`. Con `ensayo` hace todo menos publicar. |

La laptop solo tiene que estar prendida mientras corre `programar-todo` (unos minutos). A la hora puede estar apagada.

## Cada episodio

```bash
python instagram_subir.py programar-todo episodios/diaNN/diaNN.mp4 episodios/diaNN/caption.txt 19:00
#   imprime el contenedor, la hora programada y la liga del run de Actions
#   para probar sin publicar: agregar --ensayo     una URL pública propia: --url https://...
python instagram_subir.py confirmar episodios/diaNN    # después de la hora: guarda media_id y permalink
```

Por pasos (p. ej. preparar en la noche y disparar al día siguiente): `preparar <video> <caption>` y luego
`programar <carpeta> 19:00`. Portada: `IG_PORTADA_MS=<milisegundo>` antes del comando.

## Límites (y qué hace el script)

- **Ventana del workflow: 5 h 40 min.** Un job de Actions muere a las 6 h. Para las 19:00 se dispara a partir de
  las 13:20; antes, el script se niega (sin subir nada) y dice desde qué hora.
- **El contenedor vence a las 24 h.** El script se niega a programar si la hora cae más de 23 h después de preparar.
  Si de todos modos vence, el run sale en rojo (GitHub manda correo) y hay que volver a preparar.
- **Cuota de Instagram: 100 publicaciones por API cada 24 h** (ventana móvil). Crear contenedores no es publicar.
- **rupload solo sirve hasta ~55 s.** Con 58 s o más la subida directa falla (ProcessingFailedError; el contenedor
  queda con "OIL Error FILE_NOT_FOUND" y bytes_transferred 0) aunque status_code a veces diga FINISHED. No es el peso
  y subir en pedazos no sirve. Instagram necesita una URL pública: por eso el paso por `la-arena-staging`.
- **Staging:** el mp4 vive en la rama `video` solo mientras Instagram lo procesa (minutos). Al quedar listo, o si
  falla, el script empuja una rama vacía sin historia. Si ese paso falla avisa: `python instagram_subir.py limpiar`.
  Con video_url, `video_status` da "Invalid parameter" mientras procesa y luego sale `not_started` en sus fases
  aunque esté FINISHED: es normal (lo que importa es que no traiga `error`). Probado el 8-nov: Instagram baja el
  video al crear el contenedor; vaciado a los 10 s, el contenedor igual quedó FINISHED.
  GitHub no acepta archivos de más de 100 MB. Ojo: GitHub guarda un rato los commits borrados por force-push
  (se pueden abrir por su SHA); el video de todos modos se hace público a las 19:00.
- **Minutos de Actions:** el repo privado gasta los 2,000 min/mes del plan gratis y cada minuto de espera cuenta
  (disparar a las 13:20 para las 19:00 = 340 min). Disparar lo más cerca posible de la hora.

## Una sola vez (ya hecho el 8-nov)

```bash
cd "C:/Users/emili/Proyectos de programacion/Creacion de un canal de youtube con ia/la-arena-mx/publicador"
git init -b main && git add . && git commit -m "Publicador de Instagram"
gh repo create emilirex0408-dotcom/la-arena-publicador --private --source . --push
gh secret set IG_TOKEN -R emilirex0408-dotcom/la-arena-publicador < ~/.claude/instagram_token_laarenamx.txt
gh repo create emilirex0408-dotcom/la-arena-staging --public
```

`gh secret set` lee el token del archivo: nunca sale en pantalla. Cuando Meta caduque el token (~60 días), se vuelve
a correr solo esa línea. Si cambias el workflow: `git commit` y `git push` dentro de `publicador/`.
