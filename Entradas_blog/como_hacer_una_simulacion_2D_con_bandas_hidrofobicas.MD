# Introducción
## ¿De que se trata?
Básicamente se trata de un caso en el cual dejamos dos gotas de agua caer desde una altura arbitraria sobre un suelo hidrofóbico para luego estudiar y ver que es lo sucede con las gotitas. 
Tenemos gravedad y el tamaño del dominio es de 3cm x 3cm.
![condicionesSimulacion](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/como_hacer_una_simulacion_2D_con_bandas_hidrofobicas/condicionesSimulacion.png)

---
## Condiciones de la simulación
1. Es bi-dimensional o 2D.
2. Tendremos condiciones de velocidad, presión e hidrofobicidad en las paredes.
3. Tendremos dos fluidos actuando, por un lado agua. Por el otro aire.
4. No habrá temperatura y asumimos incompresible el aire.
5. El solver es el Inter-foam
6. La versión de Open Foam con la cual corremos este caso es Open Foam .org versión 9
---
# Mallado
Para el mallado utilizaremos la herramienta de Open Foam llamada `blockMesh` la cual nos permite crear un mallado bastante básico pero funcional.

Nuestro archivo que hará este trabajo se encuentra en `system/blockMeshDict` y su contenido es el siguiente:

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  9
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
FoamFile
{
    format      ascii;
    class       dictionary;
    object      blockMeshDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

convertToMeters 1;

// Variables de ancho y alto suponiendo que parten de cero
x_1 0.01;
y_1 0.03;
z_1 0.001;

x_2 0.02;
x_3 0.03;


vertices
(
    // Bloque 1
    (0  0  0)  //0
    ($x_1 0  0)  //1
    ($x_1 $y_1 0)  //2
    (0  $y_1 0)  //3
    (0  0  $z_1) //4
    ($x_1 0  $z_1) //5
    ($x_1 $y_1 $z_1) //6
    (0  $y_1 $z_1) //7

    // Bloque 2
    ($x_1 0    0)  //8
    ($x_2 0    0)  //9
    ($x_2 $y_1 0)  //10
    ($x_1 $y_1 0)  //11
    ($x_1 0    $z_1) //12
    ($x_2 0    $z_1) //13
    ($x_2 $y_1 $z_1) //14
    ($x_1 $y_1 $z_1) //15

    //Bloque 3
    ($x_2 0    0)    //16
    ($x_3 0    0)    //17
    ($x_3 $y_1 0)    //18
    ($x_2 $y_1 0)    //19
    ($x_2 0    $z_1) //20
    ($x_3 0    $z_1) //21
    ($x_3 $y_1 $z_1) //22
    ($x_2 $y_1 $z_1) //23
);

blocks
(
    hex (0 1 2 3 4 5 6 7)         (75 75 1) simpleGrading (1 5 1)
    hex (8 9 10 11 12 13 14 15)   (75 75 1) simpleGrading (1 5 1)
    hex (16 17 18 19 20 21 22 23) (75 75 1) simpleGrading (1 5 1)
);

edges
(
);

boundary
(
    inlet_hidrofobic // Suelo del cubo
    {
        type patch;
        faces
        (
            (8 9 13 12)   // Suelo del cubo 2
        );
    }
    inlet_hidrofilic // Suelo del cubo
    {
        type patch;
        faces
        (
            (1 5 4 0)     // Suelo del cubo 1
            (16 17 21 20) // Suelo del cubo 3
        );
    }
    atmosphere // Techo del cubo
    {
        type patch;
        faces
        (
            (3 7 6 2)     // Techo del cubo 1
            (11 10 14 15) // Techo del cubo 2
            (18 19 23 22) // Techo del cubo 3
        );
    }
    walls // Paredes laterales
    {
        type wall;
        faces
        (
            (0 4 7 3)     // Pared izquierda del cubo 1
            (17 18 22 21) // Pared derecha del cubo 3
        );
    }
    frontAndBack // Paredes en Z
    {
        type empty;
        faces
        (
            (0 3 2 1)     // Cubo 1
            (4 5 6 7)     // Cubo 1
            (8 9 10 11)   // Cubo 2
            (12 13 14 15) // Cubo 2
            (16 17 18 19) // Cubo 3
            (20 21 22 23) // Cubo 3
        );
    }
    interface_cubo_1_der
    {
        type wall;
        faces
        (
            (1 2 6 5)
        );
    }
    interface_cubo_2_izq
    {
        type wall;
        faces
        (
            (8 11 15 12)
        );
    }
    interface_cubo_2_der
    {
        type wall;
        faces
        (
            (9 10 13 14)
        );
    }
    inteface_cubo_3_izq
    {
        type wall;
        faces
        (
            (16 19 20 23)
        );
    }
);

mergePatchPairs
(
    (interface_cubo_1_der interface_cubo_2_izq)
    (interface_cubo_2_der inteface_cubo_3_izq)
);


// ************************************************************************* //
```

En donde estaremos definiendo tres bloques rectangulares con un mismo ancho en el eje Z. Para ello utilizaremos las variables:
- `x_1`: Siendo la dimensión en `x` donde termina el primer bloque.
- `x_2`: Siendo la dimensión en `x` donde termina el segundo bloque.
- `x_3`: Siendo la dimensión en `x` donde termina el tercer bloque.
- `y_1`: Será la altura que tendran todos los bloques (unica para todos).
- `z_1`: Ancho o grosor de nuestro sistema único para todos los bloques (esta dimensión carece de importancia debido a que es una simulación 2D).

![blockMesh](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/como_hacer_una_simulacion_2D_con_bandas_hidrofobicas/blockMesh.png)

## Refinamiento en zonas de interes
Algo tambien importante durante la fase de mallado con **BlockMesh** es que para los tres bloques estamos realizando un refinamiento en la dimensión y. Justo en la linea `hex (16 17 18 19 20 21 22 23) (75 75 1) simpleGrading (1 5 1)` lo que estamos diciendo con el `simpleGrading (1 5 1)` es que tendremos que realizar un espaciado mayor hasta 5 veces mas grande al finalizar en la dimensión y (es decir cuando `y` crece). El mallado se verá de la siguiente forma

![imagenMallado.png](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/como_hacer_una_simulacion_2D_con_bandas_hidrofobicas/imagenMallado.png)
Podemos ver que esta imagen que el mallado tiende a estar mas "junto" o "apretado" a medida que descendemos en la dirección `y`. Esto es un efecto que nos ayudará a poner mas precisión a la región del dominio donde sucederá lo que mas nos interesa. Debido a que sabemos que hay gravedad las gotas efectivamente caerán y nuestro foco deberá concentrarse en la región inferior del dominio.

## Unión de los 3 dominios o bloques creados en Blockmesh

Para realizar la unión de los 3 bloques es necesario en el diccionario boundary dentro del código de `blockMesh` definir las caras que estarán tocandose entre si, que luego tendremos que unir o "coser el mallado". Para ello tenemos este trozo de código que define las caras que se tocan de los 3 cubos, que deberán ser 4 caras. Una del primer bloque, dos del bloque del medio y una del último bloque:

```cpp
    interface_cubo_1_der
    {
        type wall;
        faces
        (
            (1 2 6 5)
        );
    }
    interface_cubo_2_izq
    {
        type wall;
        faces
        (
            (8 11 15 12)
        );
    }
    interface_cubo_2_der
    {
        type wall;
        faces
        (
            (9 10 13 14)
        );
    }
    inteface_cubo_3_izq
    {
        type wall;
        faces
        (
            (16 19 20 23)
        );
    }
```

Luego una vez definidas las caras que se estarán tocando, solo queda decir cuales son las que sus mallados se uniran, esto se logra con el diccionario llamado `mergePatchPairs`.

```cpp
mergePatchPairs
(
    (interface_cubo_1_der interface_cubo_2_izq)
    (interface_cubo_2_der inteface_cubo_3_izq)
);
```
---
# Condiciones de borde
Las condiciones de borde para este problema parten de las paredes quizá menos importantes que serán la pared izquierda, derecha y superior.

## Paredes izquierda, derecha y superior

Para estos bordes impondremos condiciones para las tres magnitudes o campos que Open Foam calculará a cada paso temporal.

### Alpha.water
Esta magnitud no es otra cosa que el porcentaje de agua que tenemos en una celda, de manera que una celda que tenga 1 del campo alpha.water tendrá un 100% de concentración de agua en su interior. Mientras que por el contrario si tenemos una celda con 0% tendremos un volumen finito sin nada de agua, o lo que es lo mismo, tendremos 100% de aire.

Para esta magnitud o campo las condiciones de contorno para estas paredes serán de `constantAlphaContactAngle`. Significa que tendremos ángulo de contacto constante a lo largo de todo el borde. En particular para estas paredes serán de 90°.

Esto lo tendremos en el archivo `0.orig/alpha.water.orig` el cual será:

```cpp
    atmosphere
    {
        type            constantAlphaContactAngle;
        theta0          90;
        limit           gradient;
        value           uniform 0;
    }

    walls
    {
        type           constantAlphaContactAngle;
        theta0         90;
        limit          gradient;
        value          uniform 0;
    }
```
En donde walls serán las paredes laterales y atmosphere será el borde superior de nuestra simulación.
 
### p_rgh (Presión dinámica)
Esta magnitud se trata nada mas que de la Presión dinámica que se define como:
$$
p\_rgh = p - \rho \cdot g \cdot h
$$
En donde $p\_rgh$ es la presión dinámica, $p$ es la presión, $g$ es la aceleración de la gravedad terrestre y $h$ es la altura. Es decir que el $p\_rgh$ no es mas que la presión quitándole el aporte de la presión estática. 

Para la presión dinámica, las paredes laterales y superior tendrán la condición de valor fijo de presión. Para el borde `atmosphere` tendremos la condición de tipo `fixedValue` y para los bordes `walls` tendremos la condición `fixedFluxPressure`.

Esto lo tendremos en el archivo `0.orig/p_rgh`:

```cpp
    atmosphere
    {
        type            fixedValue;
        value           uniform 0;
    }

    walls
    {
        type            fixedFluxPressure;
    }
```

### U (Velocidad)
El campo velocidad no es mas que la velocidad de ambos fluidos a cada paso temporal que se tenga.

Para las paredes laterales tendremos condiciones de no deslizamiento o `noSlip` que es básicamente velocidad cero en la capa límite en contacto con el borde. Mientras que para la pared superior tendremos la condición de entrada/salida de campo velocidad, comenzando con velocidad 0 en las tres dimensiones espaciales `(x, y, z)`, esta condición es `pressureInletOutletVelocity`.

Esto lo tendremos en el archivo `0.orig/U`:

```cpp
    atmosphere
    {
        type            pressureInletOutletVelocity;
        value           uniform (0 0 0);
    }

    walls
    {
        type            noSlip;
    }
```

## Bordes inferiores
¿Bordes inferiores?, si esto es debido a que justamente vamos a tener tres bordes diferentes o dicho de otra forma dos tipos de bordes por debajo.

Tal como se puede apreciar en la [[2024-11 ¿Como hacer una simulación 2D con bandas hidrofóbicas?#Esquema del sistema físico a simular|Figura 1]] en suelo del dominio físico tendremos tres secciones de suelo pintados de azul, rojo y azul consecutivamente. Esto es debido a que el suelo de tipo azul tendrá ciertas **características de borde distintas** al suelo de color rojo.

### Alpha.water
Este campo o magnitud se trata de lo anteriormente explicado en la sección [[2024-11 ¿Como hacer una simulación 2D con bandas hidrofóbicas?#Alpha.water|Alpha water]] para paredes laterales y superior.

Para los dos tipos de suelo, tendremos dos tipos de condiciones de contorno en donde variaremos la el ángulo de contacto con el suelo, esto provocará el efecto de bandas hidrofóbicas e hidrofílicas.

Para este borde las condiciones de contorno se verán de la siguiente manera:

```cpp
	inlet_hidrofobic
    {
        type            constantAlphaContactAngle;
        theta0          135;
        limit           gradient;
        value           uniform 0;
    }

    inlet_hidrofilic
    {
        type            constantAlphaContactAngle;
        theta0          45;
        limit           gradient;
        value           uniform 0;
    }
```

En donde para el bórde hidrofóbico el ángulo de contacto es de 135° y el ángulo de contacto del borde hidrofílico el ángulo será de 45°.

### p_rgh (Presión dinámica)


Para la presión dinámica, tendrá la condición de valor fijo de presión `fixedFluxPressure`.

```cpp
    inlet_hidrofobic
    {
        type            fixedFluxPressure;
    }

    inlet_hidrofilic
    {
        type            fixedFluxPressure;
    }
```

### U (Velocidad)

Para la velocidad en el suelo de bandas hidrofóbicas/hidrofílicas tendremos la condición de no deslizamiento del fluido o lo que es lo mismo velocidad cero de la capa en contacto con las paredes.

```cpp
    inlet_hidrofobic
    {
        type            noSlip;
    }

    inlet_hidrofilic
    {
        type            noSlip;
    }
```

## Paredes frontal y trasera

Para todos los campos `U`, `p_rgh` y `alpha.water` la condición de ambos bordes es de `empty`. Esto es debido a que nuestra simulación es justamente 2D, esto provoca que no tengamos ninguna modificación proveniente por parte de estos bordes hacia el fluido.

En el archivo de condiciones de contorno se verán de la siguiente manera:

```cpp
    defaultFaces
    {
        type            empty;
    }
```


---
# Seteo del campo inicial `alpha.water`

Debido a que es necesario comenzar con un fluido en alguna región del dominio que poseemos, es necesario definir en donde comenzará. Esto se hace mediante el archivo en la ruta `/system/setFieldsDict` el cual contiene la siguiente información:

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  9
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
FoamFile
{
    format      ascii;
    class       dictionary;
    location    "system";
    object      setFieldsDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

defaultFieldValues
(
    volScalarFieldValue alpha.water 0
    volVectorFieldValue U (0 0 0)
);

regions
(
    cylinderToCell
    {
        p1 (0.01 0.01 0);
        p2 (0.01 0.01 0.02);
        radius 0.002;

        fieldValues
        (
            volScalarFieldValue alpha.water 1
            volVectorFieldValue U (0 0 0)
        );
    }
    cylinderToCell
    {
        p1 (0.02 0.01 0);
        p2 (0.02 0.01 0.02);
        radius 0.002;

        fieldValues
        (
            volScalarFieldValue alpha.water 1
            volVectorFieldValue U (0 0 0)
        );
    }
);
```

Podemos ver que en seteamos por defecto los campos de `alpha.water` en `0` y la velocidad `U` en `(0 0 0)` debido a que es una magnitud vectorial. Esto quiere decir que por defecto siempre habrá aire en todas las celdas y el campo velocidad para este aire será de cero en todas las direcciones.

Sin embargo luego definimos las regiones las cuales estarán compuestas por dos cilindros atravesando el plano de nuestra simulación `cylinderToCell`. A su vez dentro de cada cilindro definimos a:
- `p1`: Como el punto central de la circunferencia inferior del cilindro.
- `p2`: Como el punto central de la circunferencia superior del cilindro.
- `radius`: Como el radio del cilindro.

![graficoSetFields](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/como_hacer_una_simulacion_2D_con_bandas_hidrofobicas/graficoSetFields.png)

---
# Archivo de arranque `Allrun`

Para el archivo de arranque `Allrun` haremos unas modificaciones simples para mejores prácticas.
En primer lugar crearemos una carpeta llamada **0.orig** en donde pondremos todas las condiciones de contorno al momento cero. Luego copiaremos esta carpeta renombrada bajo el nombre de **0** y luego si echaremos a andar la simulación.
El script de bash se verá de la siguiente forma:
```bash
#!/bin/sh
cd ${0%/*} || exit 1    # Run from this directory

# Source tutorial run functions
. $WM_PROJECT_DIR/bin/tools/RunFunctions

application=$(getApplication)

echo "Copiamos el 0.orig a 0"
cp -r 0.orig 0

runApplication blockMesh
runApplication setFields
runApplication $application

#------------------------------------------------------------------------------
```
---
# Conclusiones y links a repositorios

Lo mas importante o mas difícil del seteo de este caso es:

1. Condiciones de hidrofobicidad o ángulo de contacto.
2. Seteo de campos iniciales de la gota.
3. Creación y unión de los 3 bloques en blockMesh.

Teniendo en cuenta estos tres puntos anteriores, tenemos un caso de ejemplo excelente que abre la puerta a muchas mas situaciones similares con bandas hidrofóbicas e hidrofílicas. 

Podriamos setear diferentes tamaños de gota y formas alejadas de su forma en el equilibrio circular (por ejemplo una elipsoide). Tampoco sería dificil imaginar algún patrón de bandas hidrofóbicas distinto. Tambien podríamos setear no solo bandas, sino pilares hidrofóbicos.

Esperamos que esta explicación les haya sido de utilidad a la comunidad de los OpenFoamManiacos y les mandamos un abrazo grande.

Les dejo material importante referente a este caso:

- [Video de la simulación](https://youtu.be/1M6hwyD5w1A)
- [Carpeta del caso simulado](https://drive.google.com/drive/folders/19jLrN0KNPUASdnnTYeJwHC4DHr7qu_cD?usp=drive_link)

