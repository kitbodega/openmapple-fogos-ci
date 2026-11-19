# Compilación de `fogos` (Open Mapple) sin Mac local

Este repo pequeño contiene sólo un workflow de GitHub Actions y dos parches. No contiene
firmware, imágenes, DTB, GPT, seriales ni datos del teléfono. Las fuentes que usa son públicas
y están fijadas a un commit en el propio workflow.

## Qué hace

1. `kernel` (`macos-15-intel`): descarga `blacktop/darwin-xnu-build` en el commit fijado,
   reemplaza su `xnu/` por `open-mapple/xnu_kernel_qcom_common` + el parche FOGOS, y compila
   `MACHINE_CONFIG=FOGOS ARCH_CONFIG=ARM64 MACOS_VERSION=13.5`. El KDK 13.5 lo descarga el
   propio script desde el manifiesto público de Dortania; no hace falta cuenta de Apple.
2. `boot-image` (`ubuntu-24.04`): genera el ADT con `mkadt.py --fogos` de Open Mapple, lo
   empaqueta con el kernel usando `mkpayload.py --fogos` del shim de `ios-fogos` y construye
   un `boot.img` v3. Los dos ADT son interoperables (verificado localmente).

Salida: artefacto `fogos-boot` con `fogos-boot.img`, `Image`, `shim.elf` y `fogos.adt`.

## Cómo ejecutarlo

1. Sube este directorio a GitHub (un repo nuevo, o una rama `open-mapple-ci` de tu fork de
   `ios-fogos`). No hace falta `gh`: basta `git init && git add . && git commit`, crear el repo
   en la web y `git push`.
2. En la pestaña **Actions** del repo, habilita workflows si hace falta.
3. Ejecuta **"fogos (Open Mapple) kernel + tethered boot image"** con `Run workflow`.
   Los boot-args por defecto son **solo lectura**; no incluyen `ufsrw=1`, `rootrw=1`,
   `fogos-rw=1` ni ningún flag de escritura/borrado.
4. Descarga `fogos-boot` y arranca de forma temporal:

   ```bash
   fastboot boot fogos-boot.img
   ```

   **No** ejecutar `fastboot flash` ni tocar `userdata`, GPT, `boot`, `vendor_boot`, `dtbo`,
   `modem`, `persist` ni backups. Para volver a Android: `fastboot reboot`.

## Si falla

- El log del kernel se sube como parte del step "Build kernel" y se muestra al final en caso
  de fallo (`tail -n 250 dxb/build.log`).
- Los parches se aplican sobre commits fijos; si falla `git apply`, el upstream cambió y hay
  que regenerarlos desde el checkout local (`open-mapple/fogos/patches/`).
- `macos-15-intel` trae Xcode 16.x. El workflow elige el más antiguo disponible, igual que el
  CI actual de `ios-fogos`. Si `iig` cambia la ABI de IORPC, hará falta un runner con
  Xcode 15.4 instalado (el check de `xcrun --find iig` está en el workflow).

## Estado

Esto **compila y empaqueta**, no arranca todavía por sí solo: Open Mapple no tiene
shim/loader Qualcomm, así que se reutiliza el shim temporal de `ios-fogos` sólo para probar
el kernel y el ADT. Cada flag de hardware se debe validar en un arranque distinto antes de
seguir con el siguiente paso de la guía de porting.
