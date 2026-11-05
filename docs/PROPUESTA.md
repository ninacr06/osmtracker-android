# Propuesta: "Un punto, todos sus datos"

Propuesta del grupo para la limitación descrita en el issue #707 de OSMTracker
(datos agrupados en un único waypoint). El análisis completo y las figuras están
en el documento PDF entregado; este archivo resume la propuesta e indica dónde
se harían los cambios en el código.

**Estado: propuesta documentada, no implementada.** La rama contiene solo
cambios parciales y comentarios `TODO(propuesta)` que no rompen la compilación.
La funcionalidad completa (formulario, migración, guardado, lectura y
exportación) no está implementada.

## 1. El problema
Cuando se describe un mismo elemento con varios datos (por ejemplo, género,
especie y altura de un árbol), cada dato se registra como un waypoint
independiente.

- No existe una relación explícita que permita saber si varios puntos cercanos
  son un mismo elemento o elementos diferentes.
- Por la precisión del GPS y el movimiento de la persona, esos puntos quedan con
  coordenadas ligeramente distintas (el issue #707 lo describe).
- Al exportar, cada fila tiene un solo dato y la información queda fragmentada;
  hay que unirla a mano.
- Los conteos se inflan: tres waypoints parecen tres elementos.

## 2. Enfoque elegido: A (atributos dentro de un waypoint)
Un waypoint puede almacenar atributos opcionales como pares clave-valor
(por ejemplo, `Especie = robur`). Un único punto representa un objeto y reúne
todos sus datos.

### Flujo de interacción
1. **Iniciar la ficha:** durante la grabación, la persona presiona el botón
   «Ficha». El punto se toma de la posición GPS en ese momento, igual que hoy
   con los demás botones, y todos los atributos de la ficha quedan asociados a
   esa misma coordenada.
2. **Agregar atributos:** en el formulario «Nueva ficha» completa el nombre, una
   descripción opcional y usa «Agregar atributo» para añadir los pares
   clave-valor que necesite.
3. **Guardar:** se crea un solo waypoint con todos sus atributos.

## 3. Enfoque descartado: B (vincular varios waypoints en un grupo)
El enfoque B mantendría los waypoints separados y los asociaría con un
identificador de grupo. Se descartó porque:
- Obliga a introducir el concepto de «grupo» además del de waypoint.
- Mantiene varias coordenadas para un mismo objeto (persiste la deriva del GPS).
- Deja varias filas por objeto, que hay que unir con el identificador.
- El enfoque A refleja mejor la idea de que **un punto = un objeto**.

### Antecedente: rama `feature/form`
La rama `feature/form` de Andrew López Herrera implementa una variante del
Enfoque A con un modo de agrupación y una tabla aparte para los datos. Nuestra
propuesta la toma como referencia técnica y se diferencia en que la persona
abre una ficha por objeto y cierra la captura de forma explícita al guardar.

## 4. Cambio en los conceptos y el proceso de mapeo
- «Punto» deja de ser «un dato suelto» y pasa a ser «un objeto con sus datos».
- El levantamiento se organiza por objeto (un árbol, una banca) y no por dato.
- Vocabulario nuevo: **ficha** (registro completo de un objeto) y **atributo**
  (clave y valor). «Waypoint» queda como término técnico de GPX.

## 5. Estructura de datos propuesta
- **Almacenamiento (propuesta):** una columna nueva y opcional en la tabla de
  waypoints para guardar los atributos (por ejemplo, en formato JSON). Una
  alternativa es una tabla aparte con una fila por atributo, más fácil de
  consultar pero con una migración más compleja. Se prefiere la columna por ser
  el cambio mínimo.
- **Retrocompatibilidad:** los atributos son opcionales; los waypoints sin
  atributos se comportan como hoy.

Ejemplo ilustrativo de los atributos de un waypoint:

```json
{
  "Género": "Quercus",
  "Especie": "robur",
  "Altura": "12 m"
}
```

## 6. Exportación a GPX
GPX 1.1 solo ofrece un campo de nombre (`name`), uno de comentario (`cmt`) y uno
de descripción (`desc`) por waypoint, por lo que los atributos se escribirían en
`<extensions>`, que el estándar reserva para datos propios.

- **Compatibilidad estructural:** el archivo sigue siendo un GPX 1.1 válido,
  siempre que los elementos personalizados usen un espacio de nombres propio.
- **Compatibilidad funcional:** otras aplicaciones podrían ignorar los
  atributos, porque no conocen ese espacio de nombres.

Ejemplo ilustrativo; el espacio de nombres y los nombres de los elementos son un
ejemplo, no una decisión final:

```xml
<gpx version="1.1" creator="OSMTracker"
     xmlns="http://www.topografix.com/GPX/1/1"
     xmlns:osmt="http://ejemplo.org/osmtracker/atributos">
  <wpt lat="9.93410" lon="-84.08770">
    <name>Árbol 1</name>
    <extensions>
      <osmt:atributos>
        <osmt:atributo clave="Género" valor="Quercus"/>
        <osmt:atributo clave="Especie" valor="robur"/>
        <osmt:atributo clave="Altura" valor="12 m"/>
      </osmt:atributos>
    </extensions>
  </wpt>
</gpx>
```

## 7. Dónde se harían los cambios
Rutas desde la raíz del repositorio. `JAVA` = `app/src/main/java/net/osmtracker/`
y `RES` = `app/src/main/res/`.

| Cambio | Archivo | Responsable |
|---|---|---|
| Botón «Ficha» en la pantalla principal | `RES/layout/tracklogger.xml` y `JAVA/activity/TrackLogger.java` | Diego |
| Formulario «Nueva ficha» (nombre, descripción, lista de atributos) y botón «Agregar atributo» | Nuevo layout en `RES/layout/` y nuevo diálogo en `JAVA/view/`, tomando como modelo `JAVA/view/TextNoteDialog.java` | Diego |
| Enviar los atributos junto con el waypoint | `JAVA/OSMTracker.java` (clave del intent) y `JAVA/service/gps/GPSLogger.java` (recibe `INTENT_TRACK_WP` y guarda el waypoint) | Diego / Josué |
| Columna opcional de atributos (esquema y migración) | `JAVA/db/DatabaseHelper.java` | Josué |
| Definición de la columna en el contrato de datos | `JAVA/db/TrackContentProvider.java` (`Schema`) | Josué |
| Inserción y lectura del waypoint con sus atributos | `JAVA/db/DataHelper.java` | Josué |
| Escritura de atributos en `<extensions>` | `JAVA/gpx/ExportTrackTask.java` (`writeWayPoints`) | Alexander |
| Mostrar los atributos de un waypoint (opcional) | `RES/layout/waypointlist_item.xml` y su actividad | Diego |
| Autocompletado de claves de atributos (opcional) | Diálogo de la ficha | Alexander |

## 8. Viabilidad y riesgos
La propuesta reutiliza piezas que ya existen: el flujo de creación de waypoints,
los diálogos de captura y la escritura de `<extensions>` en la exportación GPX.
El alcance se limita a un botón y un formulario nuevos, un campo opcional con su
migración y la escritura de los atributos en el GPX.

Riesgos principales:
- Claves inconsistentes entre personas («Especie» y «especie»); mitigable con
  autocompletado.
- Aplicaciones que ignoren las extensiones del GPX.
- Conflictos de migración de base de datos si se integra con `feature/form`, ya
  que ambas modifican la base de datos.
