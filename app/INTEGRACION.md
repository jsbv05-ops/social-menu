# SOCIAL APP · Conexión para apps externas

Guía para conectar otra app (por ejemplo, la app de requisiciones de Jeank) con **SOCIAL APP**, la app del capitán de compras.

Cuando tu app envía una requisición a este endpoint, al capitán le aparece de inmediato en la pestaña **CARGA** como una "solicitud de suministro" nueva. Él marca lo que compra y lo que no hay (N/D), y cierra la carga con una nota. Tu app también puede leer ese estado para mostrar qué se compró.

- App del capitán: https://social-menu-three.vercel.app/app/
- Código: carpeta `app/` de este repositorio (`index.html`)

---

## 1. Endpoint

```
https://script.google.com/macros/s/AKfycbzvw8TvV-YjhiMt2n1akTVIOjz1z7Cdz-QzY8gDkEnF-IRFs9AOmyVAtGVVl3Wyey2c/exec
```

Es una app web de Google Apps Script. No pide token. Acepta CORS si envías el cuerpo como `text/plain`, lo que evita la petición preflight.

| Método | Qué hace |
|---|---|
| `POST` | Une lo que envías con lo que ya existe, guarda y devuelve el estado completo |
| `GET`  | Devuelve el estado completo (solo lectura) |

---

## 2. Enviar una requisición (POST)

```js
const SOCIAL_APP = "https://script.google.com/macros/s/AKfycbzvw8TvV-YjhiMt2n1akTVIOjz1z7Cdz-QzY8gDkEnF-IRFs9AOmyVAtGVVl3Wyey2c/exec";

async function enviarAlCapitan(req) {
  const r = await fetch(SOCIAL_APP, {
    method: "POST",
    headers: { "Content-Type": "text/plain;charset=utf-8" },   // importante: text/plain
    body: JSON.stringify({ state: { del: {}, reqs: [req] } })  // envía SOLO "reqs"
  });
  return r.json(); // estado completo actualizado
}

const idUnico = Date.now().toString(36) + Math.random().toString(36).slice(2, 6);

await enviarAlCapitan({
  id: "jk-" + idUnico,             // único y estable; empieza con el prefijo de tu app
  from: "Jeank · para jueves 12 nov", // título que ve el capitán
  date: new Date().toISOString(),
  sent: false,                     // false = carga abierta
  note: "",                        // déjalo vacío: es la nota de cierre del capitán
  memo: "Urgente el hielo",        // nota de quien pide (opcional)
  u: Date.now(),                   // marca de tiempo de la última edición (ms)
  items: [
    { id: "jk-" + idUnico + "-0", zone: "Nevera · Bagón 1", name: "Limones",  qty: "3 kg",     status: "pending" },
    { id: "jk-" + idUnico + "-1", zone: "Congelador",       name: "Hielo",    qty: "6 bolsas", status: "pending" }
  ]
});
```

### Campos de la requisición (`req`)

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `id` | string | sí | Único. Si reenvías el mismo `id` con una `u` mayor, se **actualiza** la requisición en lugar de duplicarla. |
| `from` | string | sí | Lo que ve el capitán como título (quién pide y para cuándo). |
| `date` | ISO string | sí | Fecha de creación. |
| `sent` | boolean | sí | Siempre `false` al crear. El capitán la pasa a `true` al cerrar la carga. |
| `note` | string | sí | Envía `""`. La llena el capitán al cerrar. |
| `memo` | string | no | Nota de quien pide; le aparece al capitán arriba del checklist. |
| `u` | number | sí | `Date.now()` del momento en que la creas o la editas. Gana la versión con la `u` más alta. |
| `items` | array | sí | Productos (ver abajo). |

### Campos de cada producto (`items[]`)

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | string | Único dentro de la requisición. |
| `zone` | string | Zona o categoría. El capitán ve los productos agrupados por este nombre. |
| `name` | string | Producto. |
| `qty` | string | Cantidad con unidad, tal cual se mostrará ("3 kg", "24", "2 botellas"). |
| `status` | string | Envía `"pending"`. El capitán lo cambia a `"bought"` (comprado) o `"na"` (no disponible). |

---

## 3. Leer el estado (GET), por ejemplo para mostrar qué se compró

```js
const estado = await (await fetch(SOCIAL_APP)).json();
const mia = estado.reqs.find(r => r.id === "jk-...");
// mia.sent        -> true cuando el capitán cerró la carga
// mia.note        -> nota de cierre del capitán ("no había Blue Curacao, llega mañana")
// mia.items[i].status -> "pending" | "bought" | "na"
```

---

## 4. Reglas

1. **Envía solo `reqs`** dentro de `state`. No envíes `crew`, `closes`, `bills`, `meetings` ni `tasks`: son la nómina, la caja y la agenda del capitán.
2. **No cambies `status`, `sent` ni `note`** de una requisición que ya existe; esos campos los maneja el capitán. Si necesitas corregir una requisición que aún no se ha cerrado, reenvíala con el mismo `id` y una `u` nueva.
3. **No borres**: deja `del` vacío (`{}`).
4. Usa un **prefijo propio** en los `id` (por ejemplo `jk-`) para que nunca choquen con los de otras fuentes.
5. El endpoint tarda de 1 a 3 segundos en responder. Los teléfonos del capitán se sincronizan cada 15 segundos o al abrir la app.

## 5. Probar rápido

```bash
curl -L -H "Content-Type: text/plain" \
  -d '{"state":{"del":{},"reqs":[{"id":"jk-prueba-1","from":"PRUEBA · borrar","date":"2026-11-11T12:00:00Z","sent":false,"note":"","memo":"solo prueba","u":1,"items":[{"id":"jk-prueba-1-0","zone":"Prueba","name":"Producto de prueba","qty":"1","status":"pending"}]}]}}' \
  "https://script.google.com/macros/s/AKfycbzvw8TvV-YjhiMt2n1akTVIOjz1z7Cdz-QzY8gDkEnF-IRFs9AOmyVAtGVVl3Wyey2c/exec"
```

Después de la prueba, avísale al capitán para que la elimine desde su app (CARGA → la solicitud → "Eliminar solicitud").
