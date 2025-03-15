# Introducción
A menudo nos enfrentamos al problema de instalar Open Foam en nuestra PC o en servidores remotos con mas recursos computacionales y por lo general este problema nos alcanza en los momentos donde mas vulnerables somos, **cuando recién empezamos**.

En este post vamos a estar viendo como instalar Open Foam en su versión `.com` para cualquier versión que queramos (Tenemos diferentes lanzamientos ej: `202312` o `202406`).

Esta forma de instalarlo nos servirá tanto para servidores remotos como para pc locales con Ubuntu o para sistemas windows pero con **WSL** (Si no sabes que es WSL, te invitamos a leer este artículo que lo explica directo de sus creadores: [Link a WSL](https://learn.microsoft.com/es-es/windows/wsl/about)).

Como consideraciones a tener en cuenta solo quiero contarte que en esta instalación **no** utilizaremos docker (ya que es posible hacer una instalación utilizándolo), ademas de que será 100% por la tan temida y amada consola utilizando lineas de comando.

En futuros post explicaré como instalarlo con esta tecnología tan interesante (docker).

# Instalación de OpenFoam.com

### Links importantes
Lo primero que deberemos tener a mano, son los links que vamos a estar utilizando durante la instalación. 

- A: [Link directo de la documentación oficial de Open Foam para instalaciones en ubuntu](https://develop.openfoam.com/Development/openfoam/-/wikis/precompiled/debian)
- B: [Página principal de Open foam .com](https://www.openfoam.com/)
- C: [Datos adicionales de la instalación](https://develop.openfoam.com/Development/openfoam/-/wikis/precompiled)

### Comenzamos la instalación

Lo que primero deberemos hacer es lanzar el siguiente comando en la consola de nuestro sistema (Abrir la terminal en nuestra pc con linux o utilizar la consola del servidor en el cual instalaremos).

```bash
# Add the repository
curl https://dl.openfoam.com/add-debian-repo.sh | sudo bash
```

Este comando agregará el repositorio de Open Foam a nuestro sistema de repositorios desde donde busca actualizaciones o busca versiones para instalar.

**Si no tenemos `curl`, nos pedirá que lo instalemos**. Curl es una herramienta instalable de los sistemas linux para hacer llamadas a URLs.

Una vez hecho esto lanzaremos el siguiente comando:

```bash
# Update the repository information
sudo apt-get update
```

Esto lo que hará es actualizar todos los repositorios, es como un especie de "refrescar" los links que tenemos en nuestro gestor de repositorios.

Posteriormente deberemos lanzar el siguiente comando:

```bash
# Install preferred package. Eg,
sudo apt-get install openfoam2406-default
```

En este punto nos podemos encontrar con el problema de que lanze que no ha encontrado el programa que queremos instalar. Si les sucede esto, recomiendo instalar la versión inmediatamente anterior a la que queríamos instalar. Por ejemplo si el comando anterior falla, les recomiendo lanzar:

```bash
# Install preferred package. Eg,
sudo apt-get install openfoam2312-default
```

Posteriormente a esto deberemos ir a la dirección en donde se ha instalado OpenFoam.com, esta dirección siempre está ubicada en el mismo lugar para nuestra fortuna. Para ir a esa ubicación lanzamos el siguiente comando:

```bash
cd /lib/openfoam/openfoam2406/
```

O con el número de la versión que instalaste. Por ejemplo si instalaste la versión 2312, deberemos poner:

```bash
cd /lib/openfoam/openfoam2312/
```

Una vez en esta dirección, estaremos en el lugar en donde se ha descargado todo el código de open foam.
### Compilando Open Foam

Como comentaba anteriormente, si bien ya hemos "instalado" open foam, lo que está sucediendo es que hemos descargado todo el código en C++ del programa, pero no lo hemos compilado. 

Esto es mas una propiedad de los lenguajes de programación compilados (no como python que es interpretado), que necesitan ser compilados para poder ser ejecutados efectivamente. Por lo tanto lo que nos falta es **compilar** open foam.

Para hacer esto, primero deberemos "cargar" las variables de entorno del programa. Hacer esto suena mas complicado de lo que realmente es, solo tenemos que lanzar el siguiente comando y **nada mas**:

```bash
source /lib/openfoam/openfoam202406/etc/bashrc
```

Una vez hicimos esto, lo siguiente será ejecutar un archivo bash que está en esa misma carpeta y que se encargará de compilarlo entero y se llama **Allwmake** (Asumo que estamos parados en la carpeta de instalación de open foam a la que llegamos en el apartado anterior).

```
└── openfoam2406
    ├── Allwmake <-- Este es el que buscamos
    ├── applications
    ├── bin
    ├── build
    ├── etc
    ├── META-INFO
    ├── platforms
    ├── src
    ├── ThirdParty
    ├── tutorials
    └── wmake
```

Lo que tiene dentro no es tan importante para nosotros, pero es un archivo bash clásico.

Ahora lo que sigue es compilar ejecutando ese archivo escribiendo:

```bash
./Allwmake -j 4 -s silent
```

Las opciones agregadas (el `-j` y `-s`) lo que harán es:
1. `-j`: Le decimos que queremos paralelizar la instalación para que sea mas veloz, luego le sigue el número de cores que vamos a asignar para su instalación.
2. `-s`: Le decimos que silencie la salida para evitar llenarnos de mensajes por consola que no son tan importantes.

Una vez finalice esto ya tendremos Open Foam instalado, felicitaciones!

**Aclaración**
Esta instalación **demora** asi que no te asustes si tarda sus buenos 45 minutos o 1 hora.

### Posible problema durante la instalación

Es muy posible (me pasó) que te enfrentes a un error extraño que no suele aparecer mucho en internet. Este error dirá algo similar a lo siguiente:

```bash
ln: fallo al crear el enlace simbólico './labelVector.H': Permiso denegado
```

No importa mucho el `./labelVector.H` ya que si te falló, seguramente lo hizo con miles de otros archivos similares. Lo importante del error es el fallo al crear el enlace simbólico.

Esto se logra resolver con el siguiente comando:

```bash
sudo chown -R tu_user_en_sistema:tu_user_en_sistema /usr/lib/openfoam/openfoam2406
```

Arrojando este comando solucionaremos el problema del enlace simbólico.

# Corremos nuestro primer caso tutorial

Para comenzar a trabajar con Open Foam y correr nuestro primer caso tutorial, recomiendo fuertemente hacer copias de los casos tutoriales que vamos a estar usando para no afectar a los originales.

Imaginando que en nuestro usuario ya creamos una carpeta para trabajar vamos a copiar un caso tutorial muy sencillo para correr y probarlo:

```bash
cp -r /lib/openfoam/openfoam2406/tutorials/multiphase/interFoam/laminar/oscillatingBox/ /home/tu_usuario/carpeta_de_trabajo
```

Dentro de la carpeta `tutorials` tenemos todos los casos ejemplo para todos los solvers. En este caso elegí uno muy sencillo de un solver llamado `interFoam`.

Una vez copiado el caso, nos metemos dentro de `carpeta_de_trabajo` que es el nombre arbitrario que elegí para mi carpeta de trabajo. Una vez dentro de la carpeta de trabajo y de la carpeta del caso, veremos algo por el estilo de carpetas y archivos:

```bash
├── oscillatingBox
│   ├── 0.orig
│   ├── Allclean
│   ├── Allrun
│   ├── Allrun.pre
│   ├── constant
│   └── system

```

Una vez dentro de esta carpeta `oscillatingBox` lanzaremos nuevamente el comando para cargar las variables de entorno:

```bash
source /lib/openfoam/openfoam202406/etc/bashrc
```

y luego para comenzar a simular este caso tutorial, lanzamos:

```bashrc
./Allrun
```

Con esto habremos corrido nuestro primer caso de simulación en Open Foam, Felicitaciones nuevamente 🥳!

# Conclusiones

Instalar open foam puede ser un proceso largo y costoso. Muchas veces nos enfrentamos a situaciones que nos desafian en cuanto a nuestro conocimiento del sistema operativo linux y sobre todo del uso de consola.

Pero no debemos desesperar, la consola y sobre todo linux son cosas ampliamente conocidas en el mundo de internet, con lo cual rapidamente podremos encontrar las soluciones a los problemas que hemos tenido y que seguramente, otras personas han tenido anteriormente.

Espero este post te haya resultado de ayuda!
