# API del backend y cómo se guardan las fotos

Todas las respuestas son JSON y llevan `Access-Control-Allow-Origin: *`, así que el frontend puede llamar al backend desde otro dominio. Los errores tienen la forma `{"error": "mensaje para mostrar"}`.

## Variables de entorno

| Servicio | Variable | Para qué |
|---|---|---|
| backend | `BUCKET` | Nombre del bucket donde se guardan las fotos. Si está vacía, el backend responde `503` |
| frontend | `BACKEND_URL` | URL del backend que utiliza el frontend |

Cloud Run proporciona además la variable `PORT`, que ambos servicios respetan.

## Endpoints

### `POST /fotos`

Recibe un formulario `multipart/form-data` con:

- `foto`: archivo de imagen.
- `autor`: nombre de quien sube la fotografía, entre 1 y 40 caracteres.

El backend abre la imagen, corrige su orientación, genera una miniatura JPEG de máximo 1024 píxeles por su lado mayor y utiliza esa miniatura para realizar dos consultas a Cloud Vision:

- `label_detection`, para obtener hasta ocho etiquetas y su confianza.
- `text_detection`, para leer el texto presente en la imagen.

El texto detectado se limita a 1.500 caracteres antes de almacenarse.

Después se guarda la fotografía original y la miniatura en Cloud Storage. La respuesta exitosa utiliza HTTP `201`:

```json
{
  "id": "20261031-234955-57317a",
  "autor": "Nelson",
  "creado": "2026-10-31T23:49:55+00:00",
  "etiquetas": [
    {
      "nombre": "Text",
      "confianza": 0.95
    },
    {
      "nombre": "Font",
      "confianza": 0.90
    }
  ],
  "texto": "PRUEBA CLOUD VISION\nFASE 6\nNELSON",
  "vision": "ok",
  "miniatura": "/fotos/20261031-234955-57317a/miniatura"
}
```

Si `label_detection` falla, la fotografía se guarda de todas formas, `etiquetas` queda como `null` y `vision` contiene un resumen del error.

Si `text_detection` no encuentra texto o falla, `texto` queda como una cadena vacía. Esto no impide guardar la fotografía ni obtener sus etiquetas.

Errores principales:

- `400`: falta el nombre, falta la fotografía o el archivo no es una imagen válida.
- `413`: la fotografía supera los 20 MB.
- `503`: el backend no tiene un bucket configurado.

### `GET /fotos`

Devuelve las últimas 24 fotografías, ordenadas desde la más reciente hasta la más antigua.

Cada elemento tiene la misma estructura utilizada por `POST /fotos`, incluyendo `etiquetas`, `texto`, `vision` y la ruta de la miniatura.

El backend obtiene la información mediante un listado de los objetos cuyo nombre comienza por `miniaturas/`.

### `GET /fotos/<id>/miniatura`

Devuelve la miniatura JPEG correspondiente a la fotografía.

La respuesta utiliza:

```text
Cache-Control: public, max-age=86400
```

Esto permite que el navegador conserve la miniatura durante un día sin volver a solicitarla.

Devuelve `404` si la miniatura no existe.

## Cómo quedan las fotos en el bucket

Por cada fotografía se almacenan dos objetos:

```text
originales/20261031-234955-57317a.jpeg
miniaturas/20261031-234955-57317a.jpg
```

`originales/` y `miniaturas/` no son carpetas reales de Cloud Storage. Son prefijos que forman parte del nombre completo de cada objeto.

La imagen original se conserva sin utilizarse en la galería. La miniatura se utiliza para mostrar la fotografía y para realizar las consultas a Cloud Vision.

## Metadatos de la miniatura

La miniatura guarda cuatro metadatos:

| Metadato | Contenido |
|---|---|
| `autor` | Nombre de quien subió la fotografía |
| `etiquetas` | Etiquetas de Vision en formato JSON, o vacío si no se obtuvieron |
| `texto` | Texto detectado por `text_detection`, con un máximo de 1.500 caracteres |
| `vision` | `ok` cuando las etiquetas se procesaron correctamente o `error: ...` cuando fallaron |

El backend nunca sirve directamente las fotografías originales. Los navegadores reciben únicamente las miniaturas.