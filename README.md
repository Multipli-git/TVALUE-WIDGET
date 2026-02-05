# TVALUE-WIDGET

Widget web en **HTML + JavaScript vanilla** para calcular y visualizar un escenario financiero usando el servicio externo de **TValue**.

La aplicación permite:
- Capturar datos del contrato (funding, pago, plazo, fecha y tasa).
- Enviar un `POST` al endpoint de cálculo de TValue.
- Mostrar resultados de flujo de caja (Cash Flow).
- Renderizar una tabla de amortización (Am Schedule) en columnas dinámicas.
- Exportar los resultados visibles a PDF.

---

## Estructura del proyecto

- `index.html`: interfaz, estilos y lógica completa del widget (UI + fetch + render + exportación PDF).
- `vercel.json`: regla de rewrite para servir `index.html` en la ruta raíz `/`.
- `README.md`: documentación del proyecto.

---

## Requisitos

No hay build ni dependencias locales.

Solo necesitas:
- Navegador moderno con soporte para `fetch`, `Promise`, `import()` dinámico y ES6.
- Conexión a internet para cargar librerías por CDN:
  - `jspdf` (incluida en `<head>`).
  - `html2canvas` (carga dinámica al exportar PDF).
- Acceso al endpoint remoto:
  - `https://tvaluerestws.com/WebApi/calculate/`

---

## Uso local

Puedes abrir `index.html` directamente en navegador, pero se recomienda servirlo por HTTP para evitar restricciones de algunos entornos.

Ejemplo con Python:

```bash
python3 -m http.server 8080
```

Luego abre:

```text
http://localhost:8080
```

---

## Flujo funcional

1. Al cargar la página, el campo **Submitted for Payout** se inicializa con la fecha actual.
2. El usuario ingresa:
   - Funding Amount Gross
   - Customer Payment Gross
   - Term
   - Submitted for Payout
   - Rate
3. Al presionar **Calculate**:
   - Se convierten montos a centavos.
   - Se arma un `CFM` con 4 eventos (loan + unknown + pagos).
   - Se construye el payload `Document` con periodo mensual y tasa anual.
   - Se hace `fetch` `POST` a TValue.
4. La respuesta se procesa para:
   - Mostrar tarjetas de Cash Flow (`#1` a `#5`).
   - Encontrar la fuente de amortización entre múltiples claves posibles.
   - Filtrar líneas no útiles (`Loan`, `Totals`, `Rounding`, etc.).
   - Mostrar pagos en tablas distribuidas por columnas según el plazo.
5. Al presionar **Export PDF**:
   - Se captura el bloque de resultados con `html2canvas`.
   - Se pagina automáticamente en PDF horizontal A4.

---

## Detalle técnico relevante

### 1) Entrada y normalización

- `funding` y `payment` se convierten a centavos (`Math.round(valor * 100)`).
- `rate` se convierte de porcentaje a decimal (`rate / 100`).
- `term` se parsea como entero.

### 2) Payload enviado al API

Se utiliza una estructura similar a:

```json
{
  "CustomerId": "6774400739051474671",
  "CreateAmSchedule": true,
  "CreateAMSchedule": true,
  "RoundingType": "SpecificLine",
  "SpecificLine": 3,
  "LineAdjustment": "LastAmount",
  "Document": {
    "CFM": [
      { "Amount": 1230000, "Event": "Loan", "Number": 1, "StartDate": "YYYY-MM-DD" },
      { "Amount": "Unknown", "Event": "Loan", "Number": 1, "StartDate": "YYYY-MM-DD" },
      { "Amount": 45000, "Event": "Payment", "Number": 35, "Period": "Monthly", "StartDate": "YYYY-MM-DD" },
      { "Amount": 45000, "Event": "Payment", "Number": 1, "StartDate": "YYYY-MM-DD" }
    ],
    "Label": "Zoho TValue Commission Only",
    "Period": "Monthly",
    "Rate": 0.0925,
    "Schema": 6,
    "YearLength": 365
  }
}
```

> Nota: el segundo elemento del `CFM` usa `Amount: "Unknown"` para que TValue calcule la comisión.

### 3) Resolución flexible de Am Schedule

La respuesta de TValue puede variar entre ambientes/versiones. Por eso el código intenta leer, en orden, distintas propiedades como:
- `AmortizationTable`
- `AmSchedule`
- `AMSchedule`
- `AmortizationSchedule`
- variantes anidadas bajo `json.CFM` o `json.Amortization`

Después intenta extraer arrays desde subclaves comunes como `Schedule`, `Items`, `Lines`, etc.

### 4) Reglas de visualización de tabla

- Se excluyen líneas de tipo resumen o redondeo.
- Solo se muestran líneas de tipo `Payment`.
- El número de columnas se calcula como `ceil(term / 12)`.
- Cada columna renderiza una tabla con: `#`, `Type`, `Amount`, `Date`, `Balance`.

### 5) Exportación PDF

- Se captura el contenedor `#parsed` con escala 2 para más nitidez.
- Se calcula slicing vertical del canvas según tamaño útil de página.
- Se agregan páginas hasta cubrir todo el contenido.
- Archivo de salida: `tvalue_result.pdf`.

---

## Despliegue en Vercel

`vercel.json` incluye:

```json
{
  "rewrites": [
    { "source": "/", "destination": "/index.html" }
  ]
}
```

Esto asegura que la raíz sirva directamente el widget.

---

## Limitaciones actuales y mejoras sugeridas

### Limitaciones
- No hay manejo explícito de errores de red/API (si falla `fetch`, no se muestra mensaje amigable).
- No hay validación de campos antes de enviar.
- Dependencia directa de un `CustomerId` fijo.
- Dependencia de CDNs externos para exportación PDF.

### Mejoras recomendadas
- Agregar validaciones de formulario y mensajes de error en UI.
- Envolver llamada a API en manejo de errores (`try/catch`) con estados de carga.
- Parametrizar `CustomerId` y endpoint por entorno.
- Separar JS/CSS en archivos dedicados para escalabilidad.
- Agregar pruebas e2e mínimas para validar render y exportación.

---

## Licencia

Este proyecto incluye archivo `LICENSE` en la raíz del repositorio.
