# Validador de túneles TP

Herramienta web de un solo archivo (`index.html`) para revisar y corregir las etiquetas de los túneles MPLS-TP estáticos en equipos **Ciena** después de hacer un bypass entre nodos (cambio de IPs en el enlace).

Compara la configuración `gmpls tp-tunnel` de los dos nodos del bypass, detecta qué túneles quedaron con etiquetas que no coinciden y genera los comandos listos para copiar en cada nodo.

Todo corre en el navegador: no hay servidor ni se envía ningún dato. Solo las fuentes (IBM Plex) se cargan de Google Fonts; sin internet se usa la fuente del sistema y la herramienta funciona igual.

## Cómo se usa

1. En cada uno de los dos nodos del bypass corre:
   ```
   conf search string tp-tunnel
   ```
2. Pega la salida completa (incluido el prompt, de ahí se toma el nombre del nodo):
   - **Nodo a corregir**: el nodo donde se van a ajustar las etiquetas.
   - **Nodo vecino del bypass**: el otro extremo del enlace nuevo. Se toma como referencia.
3. Se valida sola al pegar. Arriba aparecen los bloques **Aplicar en <nodo>** con el botón **Copiar comandos** (copia sin las líneas de comentario `!`).
4. Si quieres invertir los papeles, usa **Intercambiar nodos**.

### Opciones extra

- **IP que se quitó**: si un túnel de cualquiera de los dos nodos todavía tiene esa IP como `next-hop-ip` o `prev-hop-ip`, se reemplaza sola por la IP correcta (ver *Corrección de IP* abajo). Los que no se pueden corregir solos se listan aparte.
- **IP del nodo a corregir / IP del vecino en el enlace**: si se llenan las dos, la dirección del túnel se decide por coincidencia exacta de IPs en lugar de detectarla sola.
- **Otras etiquetas de entrada en uso**: pega aquí, por ejemplo, la salida de `conf search string in-label` para que la revisión de etiquetas ocupadas tome en cuenta servicios que no son tp-tunnel.

### Etiqueta manual

Si el equipo rechaza un túnel, abre **Detalle y etiqueta manual**, escribe la etiqueta que quieras en **Forward** o **Reverse** y presiona Enter. Los comandos de **ambos** nodos se ajustan para que sigan coincidiendo, y se avisa si ese número ya está ocupado en alguno de los dos. **Quitar manual** regresa a la propuesta automática.

## Regla de validación

Solo se comparan túneles que existen con **el mismo nombre** en los dos nodos.

### 1. Dirección del túnel (quién va antes en el camino)

Sin IPs manuales, se considera que dos IPs están en el mismo enlace si caen en la misma /30.

- Si `next-hop-ip` del nodo A y `prev-hop-ip` del nodo B están en el mismo enlace → **A es upstream** (A → B).
- Si `prev-hop-ip` de A y `next-hop-ip` de B están en el mismo enlace → **B es upstream** (B → A).
- Si no se cumple ninguna → el túnel **no pasa por el bypass** y solo se muestra como aviso.

### 1b. Corrección de IP (hop que todavía apunta a la IP quitada)

Solo se aplica si se llenó **IP que se quitó** y el túnel no pasó la regla 1.

- Si un nodo tiene la IP quitada como `next-hop-ip`, su nuevo `next-hop-ip` es **el otro extremo de la /30** del `prev-hop-ip` del vecino (en una /30, `.1` ↔ `.2`).
- Si la tiene como `prev-hop-ip`, su nuevo `prev-hop-ip` es el otro extremo de la /30 del `next-hop-ip` del vecino.
- Funciona sin importar en qué lado está el nodo con la IP vieja. El cambio se incluye en el mismo `delete` + `create`, y después se revisan las etiquetas normalmente.

Ejemplo: el nodo A tiene `prev-hop-ip 192.0.2.6` y el nodo B todavía tiene `next-hop-ip 192.0.2.81` (la IP quitada). El otro extremo de `.6` es `.5`, así que B queda con `next-hop-ip 192.0.2.5`.

### 2. Coincidencia de etiquetas

Con *up* = nodo upstream y *down* = nodo downstream:

| Sentido | Debe cumplirse |
|---|---|
| Forward | `forward-out-label` (up) = `forward-in-label` (down) |
| Reverse | `reverse-out-label` (down) = `reverse-in-label` (up) |

### 3. Corrección

- El **nodo a corregir adopta el valor del vecino** en el par que no coincide.
- Si la etiqueta que le tocaría es una **in-label** que ya usa otro túnel (u otro servicio pegado en "Otras etiquetas") en el nodo a corregir, se propone el **siguiente número libre hacia arriba** y se aplica en **los dos nodos** para que sigan coincidiendo.
- Las etiquetas que se van asignando en la misma corrida también cuentan como ocupadas, para no repetir un número entre túneles.
- Los comandos generados son `delete` + `create` con la línea completa original y solo las etiquetas cambiadas:
  ```
  gmpls tp-tunnel delete <tipo> <túnel>
  gmpls tp-tunnel create <tipo> <túnel> ... forward-in-label <nueva> ...
  ```

## Ejemplo

El botón **Cargar ejemplo** usa nombres de nodos (`NODO-A`, `NODO-B`) e IPs genéricas de documentación (`192.0.2.0/24`, `198.51.100.0/24`). Muestra un túnel por corregir con etiqueta ocupada (la reverse que le tocaría ya la usa otro túnel, así que propone la siguiente libre) y un túnel que todavía apunta a la IP quitada.

## Publicar en GitHub Pages

1. Sube `index.html` y `README.md` a la raíz del repo.
2. En el repo: **Settings → Pages → Build and deployment → Deploy from a branch**, rama `main`, carpeta `/ (root)`.
3. La herramienta queda en `https://<usuario>.github.io/<repo>/`.
