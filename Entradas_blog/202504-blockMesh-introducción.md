# Introducción

BlockMesh es una poderosa herramienta integrada dentro de **OpenFOAM** que nos permite realizar un mallado *primitivo*. Esto es debido a que se trata de un mallado mas bien de celdas de 6 caras (mallado cúbico), el cual es un mallado sencillo, sin grandes complicaciones o formas extravagantes dentro del mismo mallado. 

Esto es una ventaja y una desventaja a la vez, debido a que por un lado podemos setear un mallado de forma relativamente sencilla, por otro no podremos describir geometrías demasiado complicadas dentro de este mallado. Por ello es que se le dice *mallado primitivo*, que muchas veces alcanza y otras no.

Podríamos decir que entre todas las formas de mallar, BlockMesh es una manera primordial de crear nuestra "caja" en donde sucederá la simulación.
# ¿Donde se encuentra?

Generalmente se encuentra dentro de la carpeta `system` del arbol de carpetas de nuestro caso. 
Por ejemplo la estructura de carpetas de un típico caso de OpenFOAM se vería asi:
```bash
├── 0
│   ├── p
│   └── U
├── constant
│   └── transportProperties
└── system
    ├── blockMeshDict <---- This File is the important for this article
    ├── controlDict
    ├── decomposeParDict
    ├── fvSchemes
    ├── fvSolution
    └── PDRblockMeshDict
```

# Partes importantes del blockMeshDict

## Cabecera
Como todo archivo de OpenFOAM tendremos una cabecera en donde indicaremos que es un archivo de OpenFOAM, que es un diccionario y que objeto/tipo de diccionario es. 
```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2412                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      blockMeshDict;
}
```


## Valores del diccionario

### Scale
En primer lugar tendremos `scale`,  el cual determina las dimensiones a las que apuntaremos que la geometría esté. 

Por ejemplo, si la geometría está en milimétros nuestro valor de `scale` deberá ser de `0.001`. Luego cuando indiquemos valores específicos de las esquinas o nodos, serán pasados a milimetros. 

```cpp
scale   0.1;
```
### Vertices
Los vértices, son los puntos sobre los cuales estará montado literalmente nuestro mallado. Por ejemplo para hacer el clásico cubo tendremos que setear sus ocho vértices. Sin embargo si queremos hacer alguna otra forma complicada, o mas cubos juntos u otras formas mas complejas anexadas, deberemos establecer todos los puntos necesarios para describir la geometría deseada. 

![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/caja.png)

En nuestro caso ejemplo nuestro puntos son:
```cpp
vertices
(
    (0 0 0)   //0
    (1 0 0)   //1
    (1 1 0)   //2
    (0 1 0)   //3
    (0 0 0.1) //4
    (1 0 0.1) //5
    (1 1 0.1) //6
    (0 1 0.1) //7
);
```
### Blocks
En este apartado dentro del diccionario declararemos los bloques que conformarán a nuestro mallado. Entiéndase a cada bloque como un cuerpo físico que será mallado de manera independiente. Por ejemplo si quisieramos un cubo pegado a un tetrahedro, pues tendriamos dos bloques. En primer lugar el cubo, en segundo el tetrahedro.

Por ejemplo:
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/tetrahedro.png)

Dentro de este apartado cada bloque se declara como:
```cpp
blocks
(
	hex (0 1 2 3 4 5 6 7) (20 20 1) simpleGrading (1 1 1)       // Bloque 1
	hex (8 9 10 11 12 13 14 15) (20 20 1) simpleGrading (1 1 1) // Bloque 2
	// Mas bloques que deseemos agregar.
);
```

Para el ejemplo que estamos explicando, el caso mas sencillo del cubo, alcanzará con un solo bloque, es decir solo con `hex (0 1 2 3 4 5 6 7) (20 20 1) simpleGrading (1 1 1)`

Esto nos dá como resultado este mallado:
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/Pasted%20image%2020250420230317.png)

En donde en `hex` declaramos dentro los puntos que conforman a nuestro bloque en este caso los puntos `(0 1 2 3 4 5 6 7)` que son los que definen a nuestra caja. Luego el `(20 20 1)` está definiendo que nuestro bloque tendrá 20 celdas en `x`, 20 celdas en `y` y 1 celda en `z`. 

Posteriormente con `simpleGrading (1 1 1)` estamos diciendo que el gradiente con el cual las celdas se van a "acumular" o "espaciar" a medida que aumente esa dirección. Cada espacio refiere a una dimensión , es decir `(x y z)`. Por ejemplo si quisiéramos que todas las celdas se acumulen a la izquierda en nuestro eje X tendríamos que hacer:

```cpp
blocks
(
	hex (0 1 2 3 4 5 6 7) (20 20 1) simpleGrading (5 1 1)
);
```

Esto daría el siguiente mallado
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/mallado_izquierda.png)

En este caso estaríamos diciendo con `simpleGrading (5 1 1)` que la primer celda será 5 veces mas pequeña en X que la ultima, o lo que es lo mismo, que la ultima será 5 veces mas grande en X que la primera. 

**No trivial**
Otro punto para nada trivial es **como** seleccionamos los puntos dentro del paréntesis seguido por `hex` ya que deberá tener una cardinalidad determinada (Básicamente el orden de los puntos **si importa**). Generalmente el truco mas usado es pensarlo primero en el plano de `(x,y)` recorriendo todos los puntos para luego pasar a los puntos del plano `(x,y, 1)` o con un `z` determinado (si es que estamos hablando de una caja)

Esto lo podremos observar en el siguiente gráfico:

![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/Pasted%20image%2020250424171019.png)

### Edges

Los Edges son básicamente las lineas o las "esquinas" que unen a nuestros puntos definidos del dominio. En particular esto es opcional y podemos no incluirlo en nuestro archivo `blockMeshDict` y todo funcionará de maravilla. Sin embargo lineas que unirán a nuestros puntos serán rectas.

Si queremos describir alguna geometria con un poco mas de complejidad, por ejemplo con alguna curva en alguna de estas lineas podremos utilizar la opción de `edges`, esto se hace de la siguiente manera si seguimos el mismo caso tutorial que utilizamos en las secciones anteriores:

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

scale   0.1;

vertices
(
    (0 0 0)   //0
    (1 0 0)   //1
    (1 1 0)   //2
    (0 1 0)   //3
    (0 0 0.1) //4
    (1 0 0.1) //5
    (1 1 0.1) //6
    (0 1 0.1) //7
);

blocks
(
    hex (0 1 2 3 4 5 6 7) (20 20 1) simpleGrading (1 1 1)
);

edges
(
	arc 0 1 (0.5 0.2 0) // Agregamos una linea curva que una al nodo 0 y al nodo 1
);

```

Esto da como resultado:
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202504-blockMesh-introducci%C3%B3n/Pasted%20image%2020250424172150.png)

En **particular** la palabra clave `arc` crea un arco de circunferencia que utiliza los nodos o puntos (en este caso) `0`, `1` y el nuevo punto agregado por nosotros que sería el `(0.5 0.2 0)`.

Para los **Edges** tenemos otros tipos de curva que pueden ayudarnos, como por ejemplo:

| Palabra clave | Tipo de curva          | Datos de entrada             |
| ------------- | ---------------------- | ---------------------------- |
| arc           | Arco de circunferencia | Single interpolation point   |
| simpleSpline  | Curva spline           | List of interpolation points |
| polyLine      | Set de Lineas          | List of interpolation points |
| polySpline    | Set de splines         | List of interpolation points |
| line          | Linea Recta            | Por defecto                  |

### Boundary

Los boundary son explícitamente los bordes o límites de nuestro dominio. Podemos declararlos o no hacerlos, sin embargo si queremos diferenciar unos de otros será necesario declararlo.

Para declararlos y decir de que tipo de borde se trata, tendremos que utilizar la siguiente declaración:
```cpp
	movingWall // Nombre
    {
        type wall; // Tipo de borde
        faces // Las caras involucradas 
        (
            (3 7 6 2) // Declaramos los puntos que conforman esa cara (El orden si importa)
        );
    }
```

Dicho sea de paso los tipos de bordes que tenemos son:

| Selection Key | Description                                                                                                              |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| patch         | Borde genérico                                                                                                           |
| symmetryPlane | Plano de simetría con otro borde                                                                                         |
| empty         | Se utiliza cuando queremos anular los bordes en esa dimensión y llevar nuestra simulación a dimensiones menores, 2D o 1D |
| wedge         | cuña delantera y trasera para una geometría axi-simétrica                                                                |
| cyclic        | cyclic plane                                                                                                             |
| wall          | Simulaciones en donde nuestro borde es una pared                                                                         |
### BlockMeshDict Completo

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2412                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      blockMeshDict;
}

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

scale   0.1;

vertices
(
    (0 0 0)   //0
    (1 0 0)   //1
    (1 1 0)   //2
    (0 1 0)   //3
    (0 0 0.1) //4
    (1 0 0.1) //5
    (1 1 0.1) //6
    (0 1 0.1) //7
);

blocks
(
    hex (0 1 2 3 4 5 6 7) (20 20 1) simpleGrading (1 1 1)
);

edges
(
);

boundary
(
    inlet
    {
        type patch;
        faces
        (
            (3 7 6 2)
        );
    }
    fixedWalls
    {
        type wall;
        faces
        (
            (0 4 7 3)
            (2 6 5 1)
            (1 5 4 0)
        );
    }
    frontAndBack
    {
        type empty;
        faces
        (
            (0 3 2 1)
            (4 5 6 7)
        );
    }
);


// ************************************************************************* //
```

# ¿Como se utiliza?

Para utilizarlo es tan sencillo como pararnos en la carpeta raiz de nuestro caso y lanzar el comando `blockMeshDict`, esto correrá automaticamente nuestro caso.

**Bonus track problem**
Si te sale un cartel como este:

```bash
blockMesh
Command 'blockMesh' not found, but can be installed with:
sudo apt install openfoam
luciano@luciano-desktop:~/Desktop/ejemplo
```
y tenes instalado OpenFOAM, seguramente te faltó hacer el source a tu OpenFOAM. A esto solemos llamarlo **sourcear** OpenFOAM. Se hace de la siguiente manera (si es que tenes la versión .com de OpenFOAM)

```bash
source /lib/openfoam/openfoam2406/etc/bashrc
```

Tené en cuenta que el `openfoam202406` es porque mi versión es la v2406, si tenes otra versión deberás poner el número de la otra versión. O si tenes instalado OpenFOAM **.org** deberás ir a la carpeta `etc/bashrc` de tu versión de OpenFOAM.
# Caso Ejemplo

En este caso de ejemplo utilizaremos las configuraciones mencionadas arriba. Aún así vas a observar que no hay otros files dentro de las demas carpetas como el `0` o el `constant` y dentro de system solo vas a ver dos archivos `controlDict` y `blockMeshDict`.

Esto es en primer lugar para que puedas concentrarte solo en lo importante de este post y en segundo lugar para que entiendas como las diferentes partes de OpenFOAM trabajan algunas independientes de otras. En este caso el mallador no necesita ni condiciones de borde, ni constantes físicas ni nada que se le parezca para hacer su trabajo, es puramente geométrico.

[Link al ejemplo](https://drive.google.com/drive/folders/1pGkNn28VMjAnQ-h_ZME8xwM60yDjV6Dm?usp=sharing)

# Fuentes externas útiles

- [Documentación Oficial de OpenFOAM](https://www.openfoam.com/documentation/user-guide/4-mesh-generation-and-conversion/4.3-mesh-generation-with-the-blockmesh-utility)
- [Herramienta sumamente util para mallar sin tocar el blockMeshDict](https://visualfoam.com/)
# Conclusión

En conclusión podemos mencionar que blockMeshDict es una gran herramienta para el seteo de casos base con geometrías relativamente simples. Su ventaja es su rapidez y solidez a la hora de generar mallados sencillos, sin embargo esta también es su principal debilidad al no poder ser capaz de generar geometrías complejas.

En futuros post exploraremos el uso de SnappyHexMesh que tiene su diccionario llamado SnappyHexMeshDict el cual es clave a la hora de mallar formas mas realistas y complicadas.