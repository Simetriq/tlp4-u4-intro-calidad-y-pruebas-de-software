# Resolución — Laboratorio introductorio: pruebas automatizadas en Express.js

**Taller de Lenguajes de Programación IV · Unidad 4**

## Cambios realizados

| Archivo | Cambio |
| --- | --- |
| `demo_practica/src/services/precios.ts` | Valida que el precio sea finito y ≥ 0 y que el descuento sea finito y esté entre 0 y 100; si no, lanza `RangeError`. La fórmula se conserva. |
| `demo_practica/tests/precios.test.ts` | Pruebas nuevas: descuento 100 % → 0, precio negativo, descuento negativo y `NaN` como precio (desafío). |
| `demo_practica/tests/api.test.ts` | Prueba de integración: descuento 150 % → HTTP 400 con propiedad `error`. |
| `.github/workflows/pruebas.yml` | Se movió a la raíz del repositorio (GitHub Actions solo lee `.github/` en la raíz), con `working-directory: demo_practica`, y se cambió `npm install` por `npm ci`. |

## Cómo ejecutar

```bash
cd demo_practica
npm install
npm test
npm run typecheck
npm run dev
```

## 10. Registro de resultados

| Comprobación | Resultado | Observación |
| --- | --- | --- |
| Respuesta de `GET /healthy` | `{"estado":"ok"}` en `GET /salud` | La guía menciona `/healthy`, pero `src/app.ts` define la ruta como `/salud` (y la prueba existente también usa `/salud`). `/healthy` responde 404. |
| Primera ejecución de `npm test` | 4 pasan, 1 falla | Falla `rechaza descuentos superiores al 100 %`: *expected function to throw an error, but it didn't*. |
| Corrección del servicio | 5 / 5 pasan | Se agregaron las validaciones con `Number.isFinite` y `RangeError`. |
| Nuevas pruebas unitarias | 7 / 7 pasan | Descuento 100 % → 0; precio -100 y descuento -10 lanzan `RangeError`; `NaN` también es rechazado. |
| Prueba de integración HTTP 400 | 3 / 3 pasan | `POST /api/precios/calcular` con descuento 150 → `400 {"error":"El descuento debe estar entre 0 y 100"}`. |
| Verificación `npm run typecheck` | Sin errores | `tsc --noEmit` termina con código 0. |

Total final: **10 pruebas, todas exitosas**.

## Respuestas

### 2. Conocer la aplicación

- **¿Qué archivo contiene la regla de cálculo?** `src/services/precios.ts`, en la función `calcularPrecioFinal`.
- **¿Qué archivo define la ruta HTTP?** `src/app.ts`.
- **¿En qué se diferencian sus responsabilidades?** El servicio contiene la regla de negocio y no sabe nada de HTTP. La ruta recibe la petición, extrae `precio` y `descuento` del cuerpo, llama al servicio y traduce el resultado (o el error) en una respuesta HTTP con su código de estado.

### 4. Interpretar la falla

1. **¿Qué total devuelve un descuento del 150 % sobre $1.000?** `-500` (1000 − 1000 × 150 / 100).
2. **¿Por qué no cumple la regla de negocio?** Un descuento no puede superar el 100 % y un precio final no puede ser negativo: el sistema estaría "pagándole" al cliente.
3. **¿Por qué la prueba falla si la función terminó de ejecutarse?** Porque la prueba no exige que la función termine, sino que **lance un `RangeError`**. La función devolvió un número sin lanzar nada, así que la expectativa `toThrow(RangeError)` no se cumplió.

### 6. Desafío adicional

**¿La validación actual rechaza `NaN`?** Sí. `Number.isFinite(NaN)` devuelve `false`, por lo que se lanza `RangeError`. Sin esa comprobación, `NaN < 0` es `false` y el `NaN` pasaría la validación, devolviendo `NaN` como total.

### 7. ¿Por qué la prueba HTTP 400 es de integración?

Porque no llama a la función directamente: envía una petición HTTP real a la aplicación Express mediante Supertest. En la prueba participan el parseo del JSON (`express.json()`), la ruta `POST /api/precios/calcular`, el servicio `calcularPrecioFinal` y el manejo de errores de la ruta que convierte el `RangeError` en un `400`. Se comprueba cómo colaboran esas partes, no una unidad aislada.

### 8. ¿Por qué conviene usar `typecheck` y `npm test`?

Detectan problemas distintos. `npm run typecheck` (`tsc --noEmit`) encuentra errores de tipos sin ejecutar el código, como pasar un `string` donde se espera un `number`. Node ejecuta los `.ts` borrando los tipos, sin verificarlos. `npm test` ejecuta el código y comprueba que el **comportamiento** sea el esperado, algo que el análisis de tipos no puede verificar (por ejemplo, que un descuento de 150 % se rechace).

### 11. Preguntas de cierre

1. **Diferencia entre comprobar manualmente y ejecutar una prueba automatizada:** la comprobación manual depende de una persona que ejecuta y observa cada vez; sirve para explorar. La prueba automatizada deja la verificación escrita en código, se repite igual cada vez, en segundos, y puede correr en CI después de cada cambio.
2. **Causa de la primera prueba fallida:** `calcularPrecioFinal` no validaba sus entradas, así que con un descuento de 150 % devolvía `-500` en lugar de lanzar `RangeError`.
3. **Utilidad de incluir casos inválidos:** los errores suelen aparecer en los bordes y en las entradas inesperadas, no en el caso habitual. Probar casos inválidos verifica que el sistema rechace los datos que violan las reglas de negocio en vez de producir resultados absurdos.
4. **Qué comprueba cada archivo:** `tests/precios.test.ts` comprueba solo la función `calcularPrecioFinal` de forma aislada (unitaria). `tests/api.test.ts` comprueba la aplicación Express completa: rutas, parseo de JSON, llamada al servicio y códigos/respuestas HTTP (integración).
5. **Ventajas de ejecutar las pruebas después de cada modificación:** detectan regresiones de inmediato, cuando el cambio todavía es pequeño y fácil de identificar; dan confianza para refactorizar y, en CI, evitan que un error llegue a la rama principal sin que nadie lo note.

### Preguntas de repaso (material teórico)

1. **¿Por qué una API que devuelve HTTP 200 puede contener errores de negocio?** Porque 200 solo indica que la petición se procesó sin fallas técnicas. Antes de la corrección, la API respondía `200 {"total": -500}` ante un descuento de 150 %: técnicamente "funcionaba", pero violaba la regla de negocio.
2. **Diferencia entre prueba unitaria y de integración:** la unitaria prueba una pieza pequeña aislada (una función); la de integración prueba la colaboración entre varias piezas (ruta HTTP + servicio).
3. **¿Por qué `toThrow` recibe una función?** Para que Vitest sea quien la ejecute dentro de un `try/catch` y pueda capturar el error. Si se pasara el resultado, la llamada se ejecutaría antes, el error se lanzaría fuera de `expect` y la prueba se rompería sin poder verificarlo.
4. **Diferencia entre `npm test` y `npm run typecheck`:** `npm test` ejecuta las pruebas (verifica comportamiento); `typecheck` analiza los tipos sin ejecutar nada.
5. **¿Qué aporta GitHub Actions al trabajo colaborativo?** Ejecuta automáticamente el typecheck y las pruebas en cada `push` y pull request, en un entorno limpio. Todo el equipo ve si un cambio rompe algo antes de integrarlo, sin depender de que cada persona se acuerde de correr las pruebas localmente.
