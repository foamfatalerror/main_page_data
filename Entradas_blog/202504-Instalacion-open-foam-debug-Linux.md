
---
# Introducción
En esta oportunidad veremos como realizar una de OpenFOAM en modo **debug**. Esto significa que instalaremos una versión de OpenFOAM no productiva que nos servirá para multiples propósitos como:
1. Tocar codigo fuente de solvers.
2. Crear nuestros propios campos/solvers que resuelvan diferentes problemáticas.
3. Hacer nuestros propios cálculos de campos escalares/vectoriales.
4. Modificaciones a solvers ya existentes cuando queramos hacer alguna cosa particular.

**Observaciones**
1. En este post solo abordaremos la instalación, en post futuros estaremos hablando sobre como realizar estas modificaciones dentro de los solvers mismos.
2. Instalaremos la versión `debug` del OpenFOAM v2406.

---
# Material Adicional

 ![Video De Instalación](https://youtu.be/UMItrrbVsrA)

---
# Pasos de instalación

### ✅ Paso 1: Clonar el repositorio en otra ubicación

Elegí una carpeta donde querés instalar la nueva versión, por ejemplo si queremos hacerlo en el escritorio como fue mi caso, hacemos:

```bash
cd $HOME/usuario/Desktop
git clone https://develop.openfoam.com/Development/openfoam.git openfoam2406-debug
cd openfoam2406-debug
git checkout OpenFOAM-v2406
```

Estamos copiando el repositorio de OpenFOAM.com en una carpeta dentro de nuestro escritorio. Luego con `git checkout OpenFOAM-v2406` estamos estableciendo la versión 2406 de OpenFOAM.

---

### ✅ Paso 2: Configurar `etc/bashrc` para que compile en modo `Debug`

Editá el archivo `etc/bashrc` de esta nueva instalación, por ejemplo con:

```bash
nano etc/bashrc
```

Buscá esta línea:

```bash
export WM_COMPILE_OPTION=Opt
```

Y reemplazala por:

```bash
export WM_COMPILE_OPTION=Debug
```

---

### ✅ Paso 3: Cargar el entorno solo cuando lo necesites

Cuando necesites utilizar esta versión de OpenFOAM (no productiva) o requieras compilarla (proximos pasos) deberás lanzar este comando adaptado a tu usuario y tu escritorio..

```bash
source $HOME/usuario/Desktop/openfoam2406-debug/etc/bashrc
```

---

### ✅ Paso 4: Compilar todo en modo Debug

Una vez que tenés cargado el entorno con el `bashrc` correcto, ejecutá:

```bash
./Allwmake -j 2 -s
```

En donde las opciones:
- `-j`: Nos dice la cantidad de nucleos del procesador que va a dedicar para hacer esta instalación. En nuestro caso `2` significa que dedicaremos 2 nucleos a la compilación.
- `-s`: Nos dice que haremos la instalación en modo silenciosa, sin tantas logs.

Si querés más control, primero compilá las librerías base:

```bash
./Allwmake -j > log.make 2>&1
```

Podés revisar el `log.make` por cualquier error.

---

### ✅ Paso 5: Verificá que está compilando en modo `Debug`

Chequeá que los directorios de compilación terminan en:

```bash
platforms/linux64GccDPInt32Debug
```

En vez de `...Opt`.

### ✅ Paso 6: Probá correr cualquier caso tutorial para chequear la correcta instalación.

Intentá correr cualquier caso que desees (todos deberían de funcionar), lanzando previamente el comando:

```bash
source $HOME/usuario/Desktop/openfoam2406-debug/etc/bashrc
```

Para que lo corrra con tu versión  `debug` de OpenFOAM.

---
# Conclusión

Instalar **OpenFOAM** a veces puede ser un proceso frustrante y complejo si no entendemos que estamos haciendo, sin embargo si vamos directamente a las fuentes y respetamos prolijamente el método, deberíamos de llegar a buen puerto pues es un proceso muy conocido y estándar tanto para versiones productivas como de desarrollo.