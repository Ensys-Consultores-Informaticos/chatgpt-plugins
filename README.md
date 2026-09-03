# Gesia MCP para ChatGPT Desktop y Codex

Este directorio es una raíz de marketplace autocontenida para distribuir Gesia MCP 1.9.3. Incluye el servidor MCP local para Windows y los manifiestos que permiten descubrirlo e instalarlo desde ChatGPT Desktop y Codex.

## Qué se debe publicar

Si este directorio se convierte en un repositorio GitHub independiente, hay que subir **todo su contenido**, incluida la carpeta oculta `.agents`. En la raíz del repositorio publicado deben quedar:

```text
.agents/plugins/marketplace.json
plugins/gesia-mcp/.codex-plugin/plugin.json
plugins/gesia-mcp/.mcp.json
plugins/gesia-mcp/server/gesia_mcp-1.9.3.exe
```

No cambies esa estructura. El marketplace resuelve `./plugins/gesia-mcp` desde la raíz del repositorio.

## Probar antes de publicar

Desde PowerShell, registra este directorio local como marketplace:

```powershell
codex plugin marketplace add "C:\RUTA\AL\REPO\chatgpt-plugins"
codex plugin marketplace list
```

Reinicia ChatGPT Desktop. Después abre el directorio de plugins, selecciona la fuente **Gesia Plugins** e instala **Gesia MCP**. Haz la prueba en un chat nuevo.

Para eliminar posteriormente esa fuente local:

```powershell
codex plugin marketplace remove gesia-plugins
```

## Instalar desde GitHub

Una vez publicado el contenido de este directorio como raíz de un repositorio:

```powershell
codex plugin marketplace add PROPIETARIO/REPOSITORIO
codex plugin marketplace list
```

Reinicia ChatGPT Desktop, abre el directorio de plugins, selecciona **Gesia Plugins** e instala **Gesia MCP**. Para incorporar versiones posteriores del repositorio:

```powershell
codex plugin marketplace upgrade gesia-plugins
```

## Configurar y usar Gesia

La distribución no incluye una ruta `GS3_FILE`, porque cada usuario y cada expediente tienen una ruta distinta. Usa la herramienta MCP `configurar` al comenzar la sesión para indicar el expediente `.gs3`. Si se desea una ruta fija, puede configurarse localmente mediante `GS3_FILE`; nunca debe incorporarse esa ruta personal al repositorio.

Antes de consultar datos:

1. Abre el expediente en Gesia.
2. Arranca su API desde **Herramientas > Gesia - Cuadro de mando > Arrancar servidor API**.
3. Solo para consultar diarios `.smn`, instala Microsoft Access Database Engine 2016 de 64 bits. Las consultas al expediente `.gs3` no necesitan ese componente.

El servidor expone siete herramientas: `configurar`, `contexto_expediente`, `consultar_gesia`, `consultar_diario`, `obtener_entidad`, `exportar_consulta` y `limpiar_exportaciones`.

`exportar_consulta` deja el resultado de una consulta en un fichero en vez de devolverlo, que es como se trabaja con volúmenes grandes sin traérselos al chat. Esos ficheros llevan la contabilidad del cliente, así que **al terminar hay que llamar a `limpiar_exportaciones`**, que borra los que el propio servidor ha escrito. Si no se llama, se quedan en el disco.

## Procedimientos incluidos

Además del servidor MCP, el plugin trae siete procedimientos de auditoría, entre ellos las dos pruebas de muestreo de ForSampling (cumplimiento y MUM) contra las facturas escaneadas. Cada uno lee el
expediente por el MCP, calcula con sus propios scripts y deja el papel de trabajo en
`InformesGesia` dentro del expediente. Son **propuestas de papel de trabajo**: la
valoración, el alcance y la conclusión son del auditor que firma.

| Procedimiento | Qué hace |
|---|---|
| `continuidad-saldos` | Compara la apertura del diario con los saldos auditados del ejercicio anterior |
| `cancelacion-saldos` | Empareja facturas con sus pagos y cobros, y deja a la vista el saldo vivo |
| `cuestionario-cuentas-anuales` | Revisa qué desgloses faltan en la memoria, leyendo las cuentas anuales en PDF |
| `identificacion-riesgos` | Elige los riesgos del catálogo del máster conforme a la NIA-ES 315 |
| `registro-ejecucion` | Escribe en el chat cómo ha ido la ejecución, para reportar incidencias |

Dos procedimientos que sí están en el canal de Claude **no se distribuyen aquí**: el cuadro
de mando del diario, pendiente de rehacerse, y la investigación de la entidad en fuentes
públicas, que necesita subagentes y Codex no los tiene.

## Límite importante: ChatGPT Desktop frente a ChatGPT web

Este plugin contiene un ejecutable Windows y usa MCP mediante `stdio`. Por tanto, funciona en superficies locales capaces de ejecutar el servidor, como ChatGPT Desktop y Codex en Windows.

Subir estos archivos a GitHub **no convierte el ejecutable en un servidor accesible por ChatGPT web**. Para usarlo en `chatgpt.com` se necesita exponer Gesia MCP mediante un endpoint MCP HTTPS público o un túnel seguro, registrar esa conexión en el modo desarrollador de ChatGPT y empaquetar su identificador en un archivo `.app.json`. Ese despliegue no forma parte de esta distribución.

## Diagnóstico

- Si el plugin no aparece, confirma que `.agents/plugins/marketplace.json` está en la raíz publicada, ejecuta `codex plugin marketplace list` y reinicia ChatGPT Desktop.
- Si el servidor no arranca, comprueba que Windows no haya bloqueado `plugins/gesia-mcp/server/gesia_mcp-1.9.3.exe` y verifica su SHA-256 con `Get-FileHash`.
- Si no hay conexión con Gesia, abre el expediente y vuelve a arrancar el servidor API desde el menú de Gesia.
- Si no encuentra el expediente, llama a `configurar` con la ruta correcta o revisa la variable local `GS3_FILE`.
- Si solo fallan los diarios `.smn`, revisa Microsoft Access Database Engine 2016 de 64 bits.
- Si una actualización no se refleja, ejecuta `codex plugin marketplace upgrade gesia-plugins`, reinicia la aplicación y abre un chat nuevo.

## Seguridad y publicación

No publiques expedientes `.gs3`, diarios `.smn`, archivos `config.toml`, variables de entorno, credenciales, rutas personales ni datos de clientes.

`gesia_mcp-1.9.3.exe` es software propietario. Antes de hacer público el repositorio, confirma expresamente que existe autorización para redistribuir el ejecutable. El hash esperado del binario incluido figura en `plugins/gesia-mcp/SHA256SUMS.txt`.
