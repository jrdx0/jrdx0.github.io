---
title: 'Flameshot: Arreglar icono del tray en COSMIC DE'
description: 'Si eres usuario de COSMIC DE y de Flameshot es posible que te hayas encontrado con un problema que, pese a no afectar directamente al funcionamiento de la aplicación ni al sistema operativo como tal, puede llegar a ser incómodo para la vista. El icono de Flameshot no se muestra correctamente en el panel superior de COSMIC. En esta guía podrás encontrar la solución que me ha funcionado para corregir dicho error.'
pubDate: '2026-04-19'
---

> Si quieres ir directo a la guía paso a paso para solucionar el error, [haz clic aquí](#solución-guía-paso-a-paso). Si quieren saber más sobre el contexto, sigan leyendo.

Si bien no es un problema que afecte de forma critica al sistema operativo o al funcionamiento de mi equipo, era imposible evitar mirar ese icono gris horrible. Desentonaba mucho. Así que un fin de semana me senté a intentar solucionar el problema. Fueron dos días de investigar cómo hacer que COSMIC mostrara correctamente el icono que debía mostrar, y después de varias pestañas del navegador abiertas, pláticas extensas con CLAUDE y cuestionarme en repetidas ocasiones si de verdad valía el tiempo y esfuerzo; lo pude lograr.

## ¿Qué sucedía?

Por temas de compatibilidad tuve que instalar Flameshot desde la COSMIC Store, la cual hace uso de Flatpak para realizar las instalaciones de las aplicaciones. Hasta el día de hoy no es posible la instalación de Flameshot por `apt`.

Es con Flatpak donde surge el error de no mostrar correctamente el icono en el tray (panel superior de COSMIC), ya que Flatpak no exporta correctamente dicho icono por defecto. La solución consiste en crear un symlink al icono correcto dentro de una carpeta donde COSMIC sí pueda encontrarlo.

## Solución: Guía paso a paso

Los primeros 4 pasos nos ayudarán a localizar el nombre que el tray de COSMIC espera encontrar y donde se encuentra el archivo original dentro de la instalación de Flameshot.

### 1. Verificar que Flameshot está registrado en el tray

Antes que nada debemos tener Flameshot corriendo en nuestra tray (si vemos el icono feo del engrane y al darle clic se nos abre la GUI para hacer capturas de pantalla, vamos bien. Por lo tanto hay que verificar que Flameshot se ha registrado en los whatchers de COSMIC.

```bash
busctl --user get-property org.kde.StatusNotifierWatcher /StatusNotifierWatcher org.kde.StatusNotifierWatcher RegisteredStatusNotifierItems
```

Al ejecutar dicho comando esperamos obtener una lista de bus names. En mi caso:

```
NotifierItems
as 3 ":1.149" ":1.151" ":1.159"
```

Si en tu caso no aparece la información deseada, lo siento pero los siguientes pasos no funcionarán. Puede que tu problema no sea tema de los iconos.

### 2. Localizar el objeto D-Bus de Flameshot

[D-Bus](https://www.freedesktop.org/wiki/Software/dbus/) en Linux es un sistema de mensajería. Básicamente, al ejecutar Flameshot, éste se conecta a un D-Bus para exponer un objeto que le dirá al tray de COSMIC dónde puede encontrar los recursos para mostrarse correctamente en el panel.

Vamos a enlistar aquellos que estén relacionados con flameshot con el siguiente comando:

```bash
busctl --user list | grep -i flameshot
```

El comando imprimirá una lista de bus names de la forma `:1.XXX` correspondientes a Flameshot:

```
:1.124                 5248 flameshot  userName :1.124  user@1000.service -  -
:1.149                 5248 flameshot  userName :1.149  user@1000.service -  -
org.flameshot.Flameshot  5248 flameshot  userName :1.124  user@1000.service -  -
```

Ahora inspeccionaremos su árbol de objetos para confirmar que `/StatusNotifierItem` existe:

```bash
busctl --user tree :1.149   # sustituye :1.149 por tu bus name
```

Si todo sale bien, podremos confirmar la existencia de `/StatusNotifierItem` por el siguiente output:

```
├─ /MenuBar
└─ /StatusNotifierItem
```

### 3. Obtener el valor de IconName que anuncia Flameshot

Aquí empieza la parte importante. Es necesario saber el nombre real del icono que Flameshot anuncia. Este paso es sumamente importante pues utilizaremos dicho valor para crear el icono nosotros mismos. Recuerden sustituir el valor del bus name por el que hayan obtenido en el [paso 2](#2-localizar-el-objeto-d-bus-de-flameshot):

```bash
busctl --user get-property :1.149 /StatusNotifierItem org.kde.StatusNotifierItem IconName
```

Obtendremos un output anunciando el verdadero nombre del icono:

```
s "flameshot-tray"
```

¡Perfecto! Ya tenemos todo para empezar a solucionar el problema.

### 4. Localizar el SVG correcto dentro de Flatpak

Como dije en un principio, por alguna extraña razón Flatpak no expone el icono correcto para que COSMIC lo muestre en el panel superior. Es necesario que lo localicemos nosotros mismos dentro de los archivos de instalación.

Para ello hay que localizar la carpeta donde Flatpak gestiona todos sus archivos, en mi caso: `~/.local/share/flatpak`. Ahora es momento de buscar el nombre del icono que no se exporta correctamente:

```bash
find ~/.local/share/flatpak -name "*flameshot-tray*" 2>/dev/null
```

El comando `find` enlistará todos los archivos cuyo nombre tenga `flameshot-tray` sin importar el contenido antes y después de dicho nombre. Se enlistarán las rutas de los archivos encontrados:

```
~/.local/share/flatpak/runtime/org.kde.Platform/x86_64/6.9/<hash>/files/share/icons/breeze/status/22/flameshot-tray.svg
~/.local/share/flatpak/runtime/org.kde.Platform/x86_64/6.9/<hash>/files/share/icons/breeze/status/22/flameshot-tray-symbolic.svg
```

La ruta que se te muestra puede ser más larga debido a que he acortado el valor para el hash de la carpeta con `<hash>`. Si somos más curiosos podremos observar que en la carpeta `~/.local/share/flatpak/runtime/org.kde.Platform/x86_64/6.9/` podremos encontrar un symlink que apunta al directorio del `<hash>`. Esto nos puede ayudar para mantener algo de persistencia en caso de que el `hash` llegue a cambiar. Puedes confirmar esto ejecutando (recuerda cambiar la ruta para tu caso):

```bash
ls -l ~/.local/share/flatpak/runtime/org.kde.Platform/x86_64/6.9/
```

Obtendremos algo como lo siguiente, donde el symlink que nos interesa se llama `active`:

```
lrwxrwxrwx 1 userName userName   64 Feb 26 16:42 active -> <hash>
drwxr-xr-x 3 userName userName 4096 Feb 26 16:42 <hash>
```

### 5. Crear un symlink donde COSMIC pueda encontrarlo

Ahora toca crear un symlink al archivo que encontramos en el [paso anterior](#4-localizar-el-svg-correcto-dentro-de-flatpak). Un lugar en el que es visible y no es necesario entrar en demasiados directorios es la ruta `~/.local/share/icons/hicolor`. Navegando a dicha carpeta encontraremos diferentes directorios que agrupan los iconos según su tamaño. Al tratarse de un SVG, podemos agregar el icono de Flameshot dentro del directorio `scalable`. Dentro del mismo habrá que crear una carpeta llamada `status`:

```bash
mkdir -p ~/.local/share/icons/hicolor/scalable/status
```

Posteriormente crearemos la referencia al archivo original con ayuda de un symlink:

```bash
ln -sf \
  ~/.local/share/flatpak/runtime/org.kde.Platform/x86_64/6.9/active/files/share/icons/breeze/status/22/flameshot-tray.svg \
  ~/.local/share/icons/hicolor/scalable/status/flameshot-tray.svg
```

Listo. Problema solucionado. Solo queda recargar el caché de iconos y comprobar que todo ha funcionado.

### 6. Reconstruir el caché y reiniciar el panel

Pese a que los archivos se encuentran en el lugar correcto, aún no se deberían de mostrar en el panel. COSMIC cachea los iconos, por lo que es necesario recargar dicho caché para que obtenga el que hemos creado nosotros mismos. Para ello es necesario ejecutar el siguiente comando:

```bash
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor
```

Finalmente hay que reiniciar el panel para comprobar si todo ha salido bien. Para no tener que reiniciar la computadora entera, basta con matar el proceso del panel y COSMIC lo arrancará de manera automática por nosotros:

```bash
pkill cosmic-panel
```

Ahora deberíamos dejar de ver el icono del engrane gris y mirar el icono correspondiente de flameshot.

### Conclusión

Es lo que tiene Linux: a veces solucionar un problema menor requiere más investigación de la esperada, pero la gratificación que produce lograrlo lo vale. Espero que esta guía le haya sido de utilidad a alguien.
