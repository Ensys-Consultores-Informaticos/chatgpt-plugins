---
name: continuidad-saldos-gesia
description: >
  Prueba de continuidad de saldos de apertura de un expediente de auditoría de
  Gesia. Compara cuenta por cuenta la apertura del diario normalizado (.smn) con
  los saldos finales auditados del ejercicio anterior (.gs3), al nivel de
  auditoría del expediente, y produce el papel de trabajo en Excel.
  Usa este skill cuando el usuario pida comprobar o cuadrar los saldos de
  apertura, la continuidad de saldos, la prueba de apertura, el AG)02, o diga
  cosas como "cuadra la apertura", "comprueba los saldos de apertura", "prueba
  de continuidad", "revisa que la apertura casa con el cierre auditado" o
  "saldos iniciales del ejercicio".
  NO es el cuadro de mando del diario ni el cuestionario de la memoria.
  Requiere Gesia abierto con el servidor API arrancado, un expediente con al
  menos dos ejercicios y diario importado.
---

# Continuidad de saldos de apertura (Gesia)

Compara lo que el cliente ha abierto con lo que quedó auditado al cierre
anterior. La diferencia es el hallazgo; cero en todas las cuentas es el
resultado esperado.

Los ficheros se escriben en `<expediente>\InformesGesia\ContinuidadSaldos\` y
**nunca se publican** en ninguna URL: son la contabilidad de un cliente
auditado.

---

## Cuatro reglas que gobiernan todo

**1. El asiento de apertura no se adivina.** Se fija al normalizar el diario y no
tiene que ser el número 1 —en el expediente de calibración era el 3495—. Vive en la tabla
`CfgNormalizacion` del `.smn`, en el registro `FiltroDeApertura`. Si ese filtro
no se puede leer o trae un operador desconocido, **el skill para**. Elegir el
asiento por intuición da un resultado plausible y falso.

**2. La comparación va al nivel de auditoría del expediente que se audita,** no
al del papel del año pasado ni al desglose del cliente. Y ese nivel **no es un
número fijo de dígitos**: en el expediente de calibración hay 204 cuentas de
máximo nivel a 3 dígitos y 234 a 4. Cada cuenta del diario se asigna a la cuenta de máximo nivel
cuyo código sea **el prefijo más largo** que exista.

**3. Solo grupos 1 a 5.** Los grupos 6 y 7 se regularizan contra resultados y no
tienen saldo de apertura; el 0, el 8 y el 9 quedan fuera. En el expediente eso es
`CodigoTipo = 'B'`, que es más fiable que mirar el primer dígito.

**4. Una cuenta a cero en los dos lados no sale en el papel.** Son cuentas del
plan que el cliente no usa —389 de 438 en el expediente de calibración— y solo
estorban. El
recuento de omitidas se dice en pantalla: el recorte no es silencioso.

---

## Dónde están los scripts

Los scripts están en `skills/continuidad-saldos/scripts/`, dentro de la raíz del
plugin instalado.

**Sustituye los marcadores de los ejemplos por rutas reales antes de ejecutar.**
No uses variables de shell: en Windows el intérprete no siempre es el mismo y una
variable que funciona en una consola no existe en la otra.

- `<PLUGIN>` — la raíz del plugin instalado
- `<TRABAJO>` — un directorio de trabajo que creas tú, **fuera del expediente**

El diario se lee por el **driver ODBC de Access de 64 bits**, el mismo que usa el
MCP. Los scripts abren el `.smn` donde está, sin copiarlo ni convertirlo, y no hay
que indicarles nada.

---

## Secuencia

### Paso 1 — Expediente

```
configurar()            # sin parámetros: ver estado
contexto_expediente()
```

Si falta `gs3_file`, **pide la ruta**. Si el servidor API no responde, dile que lo
arranque en *Herramientas > Gesia - Cuadro de mando > Arrancar servidor API*. Si
el expediente no tiene diario importado, **para**: sin diario no hay apertura que
comparar, y eso no se arregla indicando otra ruta.

### Paso 2 — El cierre auditado del ejercicio anterior

El `.gs3` es un `.mdb` con contraseña y **solo se lee por el API**, así que este
dato no se puede leer desde el contenedor.

**Se exporta a fichero, no se trae al contexto.** `exportar_consulta` ejecuta la
consulta y deja el resultado en disco; tú solo ves cuántas filas ha escrito. Son
unas 440 y antes costaban del orden de 7.000 tokens al traerlas y otros tantos al
volver a escribirlas.

```
exportar_consulta(
  sql = "SELECT ca.Cuenta, c.Nombre, ca.SaldoAuditoria
         FROM CuentasAuditorias AS ca LEFT JOIN Cuentas AS c ON ca.Cuenta = c.Cuenta
         WHERE ca.CodigoAuditoria = '<anterior>'
           AND ca.MaximoNivel = True AND ca.CodigoTipo = 'B'",
  ruta = "<TEMP>/gesia-continuidad/cierre.json")
```

El `CodigoAuditoria` es un orden interno, **no un año**: sácalo de `Auditorias`
por el campo `Año` y no lo calcules sumando uno.

**No abras ese fichero.** Los scripts lo leen tal cual: aceptan las filas crudas
del export y solo necesitan que les digas los ejercicios, que la consulta no trae.

### Paso 3 — El diario

**El único diario válido es el que el expediente tiene vinculado**, el que declara
`configurar` / `DatosAuditoria`. No hay nada que decidir: se usa ese.

En la carpeta puede haber otros `.smn` —dos o cuarenta, es normal— y **no se
miran**. Ni se listan, ni se comparan fechas, ni se ofrecen como alternativa. Que
uno parezca más nuevo o tenga mejor pinta no lo convierte en el diario del
encargo.

Dos excepciones, y solo dos:

- **No hay diario vinculado** → **para**. Sin diario no hay apertura que comparar,
  y no se arregla cogiendo un `.smn` parecido de la carpeta. La única salida es que
  el usuario aporte el diario en ese momento: pídeselo.
- **El usuario dice que el vinculado no vale** y señala otro → se usa el que él
  diga, y el papel de trabajo deja constancia de que no es el vinculado.

Por qué se insiste tanto: medido en cliente el 23/08/2026, un expediente tenía dos
`.smn` que **daban resultados contradictorios** — con uno la apertura salía
perfecta y con el otro aparecía un hallazgo de 4,78 M€. Si el resultado sorprende,
la pregunta no es «¿habrá otro fichero mejor?», es si lo que Gesia tiene vinculado
es lo que el auditor cree. Y eso se arregla en Gesia reimportando, no eligiendo
otro fichero aquí.

En la máquina del auditor no hay nada más que hacer: los scripts abren el `.smn`
donde está, por ODBC.

Todo corre en la máquina del auditor, así que **no hay nada que subir ni que
convertir**: pásale al script la ruta del `.smn` tal como la devuelve `configurar`.

Lo que sí importa es dónde dejas lo que escribas. Cualquier fichero que generes
con `exportar_consulta`, y el directorio de trabajo, van a la carpeta temporal del
sistema o a `<TRABAJO>` — **nunca dentro de la carpeta del expediente**. Ahí van
los papeles de trabajo, y contabilidad del cliente suelta al lado del papel
firmado no es un descuido estético: es lo que el auditor archiva.

### Paso 4 — Verificar el contrato (puede abortar)

```bash
python "<PLUGIN>/skills/continuidad-saldos/scripts/verificar_contrato.py" --diario "<diario>" \
    --cierre "<TEMP>/gesia-continuidad/cierre.json" \
    --ejercicio 2025 --ejercicio-anterior 2024 --cliente "<razón social>"
```

Salida `2` → **para**, no se genera nada. Salida `1` son avisos: léelos y
**cuéntalos al entregar**, porque cambian lo que el papel puede concluir. Los dos
que más importan: que el campo `APERTURA` no coincida con el filtro (suele
significar que el filtro se cambió después de importar) y que la apertura traiga
cuentas de los grupos 6 y 7.

### Paso 5 — Comparar

```bash
python "<PLUGIN>/skills/continuidad-saldos/scripts/comparar.py" --diario "<diario>" \
    --cierre "<TEMP>/gesia-continuidad/cierre.json" \
    --ejercicio 2025 --ejercicio-anterior 2024 --cliente "<razón social>" \
    --salida "<TRABAJO>/resultado.json"
```

Imprime el resumen y los hallazgos. **No leas `resultado.json` entero**: el
resumen de pantalla ya trae lo que hay que contar.

Tres clases de hallazgo, y no son lo mismo:

| Tipo | Qué pasa |
|---|---|
| `diferencia` | la cuenta está en los dos lados y el importe no cuadra |
| `sin_apertura` | tenía saldo al cierre y el cliente no la ha abierto |
| `cuenta_nueva` | se ha abierto con saldo y el expediente no la reconoce a ningún nivel |

**Y mira el cuadre de control, que es la línea más importante de la salida.** La
apertura de balance tiene que sumar cero. Si no suma cero, lo que falta es
exactamente lo que se ha quedado fuera de la comparación —patas en cuentas de
resultados y cuentas no reconocidas— y el script lo desglosa. Si queda algo
**sin explicar**, no entregues el papel: hay algo que el skill no está
entendiendo.

Medido en el fixture de pruebas: descuadre de 383.333, que son 333.333 de una
cuenta 710 metida en el asiento de apertura más 50.000 de una cuenta nueva. Sin
esta comprobación el papel salía diciendo que la caja no cuadraba, sin mencionar
que había un ingreso dentro de la apertura.

### Paso 6 — El papel de trabajo

```bash
python "<PLUGIN>/skills/continuidad-saldos/scripts/generar_papel.py" --resultado "<TRABAJO>/resultado.json" \
    --fecha-cierre "31/12/25" \
    --salida "<expediente>/InformesGesia/ContinuidadSaldos/AG)02 Saldos de Apertura <CLIENTE> <EJERCICIO>.xlsx"
```

`--fecha-cierre` es como se escribe en la cabecera del papel y **lo pones tú**:
nada en este skill lee el reloj. Sale de `FechaCierre` de `contexto_expediente`.

La referencia es `AG)02` («ESTIMACIONES Y SALDOS APERTURA CONTABLES»), que es la
que tiene el expediente. `REF`, `Realizado` y `Verificado` van en blanco a
propósito: los rellena el auditor.

### Paso 7 — Entregar

Di dónde ha quedado el fichero, **cuántas cuentas lleva y cuántos hallazgos de
cada clase**. Si no hay ninguno, dilo así: la continuidad está comprobada y no
hay diferencias. Y recuérdale que:

- el papel es una propuesta y la revisión y la firma son suyas.

**Los temporales.** Lo que haya escrito `exportar_consulta` lo borra
`limpiar_exportaciones()`, y esa es la vía: no hay que decirle qué fichero,
porque borra lo que él mismo escribió. Si algo no se deja borrar (un `.csv`
abierto en Excel, típicamente), lo dice con su ruta: trasládala al usuario.

El directorio de trabajo va aparte y lo borras tú: elimina `<TRABAJO>` con la
orden que corresponda a la consola en la que estés.

Si falla, **di las rutas exactas** de lo que queda para que el usuario lo quite.
Nada de dejarlo caer: son datos de su cliente. Por eso, además, los temporales
van a la carpeta temporal del sistema y no al expediente — ahí los limpia
Windows y no estorban entre los papeles de trabajo.

---

## Lo que este skill no hace

- **No escribe en el expediente.** El MCP solo lee.
- **No explica la diferencia**, solo la localiza. Si una cuenta no cuadra, por qué
  no cuadra lo averigua el auditor.
- **No comprueba que el cierre anterior esté bien auditado.** Toma
  `SaldoAuditoria` del ejercicio anterior tal como está en el expediente, con sus
  ajustes ya incorporados.
- **No mira los grupos 6 y 7**, ni el 0, ni el 8, ni el 9.
- **No vale para un encargo inicial** tal cual: la NIA-ES 510 se aplica cuando el
  cierre anterior **no** lo auditamos nosotros, y entonces este cruce no es
  evidencia suficiente. El cálculo sirve igual; lo que cambia es qué puede
  concluir el papel.

## Degradación

| Situación | Qué sale |
|---|---|
| Sin diario importado | **para.** No hay apertura que comparar |
| Sin `CfgNormalizacion` o sin `FiltroDeApertura` | **para.** El asiento de apertura no se adivina |
| Operador de filtro desconocido | **para**, y dice cuál era |
| La apertura no cuadra | **para.** Con la apertura descuadrada la prueba no significa nada |
| Campo `APERTURA` vacío | sigue con el filtro, y avisa de que pierde la comprobación cruzada |
| Expediente con un solo ejercicio | **para.** No hay cierre anterior |
| Cuentas abiertas sin destino en el expediente | sigue, y las emite como hallazgo `cuenta_nueva` |
| La apertura de balance no cuadra | sigue, y el papel lo avisa arriba con el desglose |
