# Introducción
## ¿De que se trata?

En esta ocasión estaremos realizando una serie de simulaciones muy ligeras (ya que son 2D y con muy pocas celdas) pero de una manera automatizada aprovechando de la potencia de Linux y sus scripts de **bash**.
La idea es poder dejar corriendo un solo script de bash y que este mismo sea quien ejecute una serie de simulaciones, una tras otra, hasta llegar a la cantidad deseada.
## Consideraciones básicas iniciales

1. Estaremos trabajando en Open Foam 9 de la distribución .org
2. Estaremos trabajando con una simulación Bi-Dimensional y con pocas celdas (ligera).
3. Estaremos trabajando en sistemas con Linux (Específicamente Ubuntu) ya que haremos un script de bash para automatizarlo todo.
4. Utilizaremos el Solver **interFoam**.
5. A lo largo de las simulaciones variaremos dos magnitudes:
   - Angulo de contacto de una superficie.
   - Radio de ambas gotas del experimento (Ver figura 1 de la sección **Sobre la simulación**).
---
# Sobre la simulación

## Solver + situación física
Como antes habíamos mencionado el solver que utilizaremos es el interFoam ya que lo que deseamos lograr es una simulación de dos gotas uniéndose y este solver nos aporta todo lo necesario como por ejemplo, ángulos de contacto, formaciones de gota, interfaces entre fluidos de distintas características (aire y agua).
La situación física que vamos a estar simulando es la siguiente:



![imagen de experimento](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/como_hacer_una_simulacion_automatizada_en_open_foam_y_linux/figura%201%20-%20experimento.png)

En donde el ángulo de contacto variable será el que iremos moviendo simulación a simulación. Lo mismo aplicará a el radio de ambas gotas.

---
## Dominio y condiciones de contorno + iniciales

### Archivo BlockMesh de nuestro caso ejemplo
Para la creación de nuestro dominio en blockMesh utilizaremos 3 bloques contiguos y pegados, en donde en cada uno iremos declarando las superficies para poder setear sus propiedades en los archivos de condiciones de contorno.

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
y_1 0.004;
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
    hex (0 1 2 3 4 5 6 7)         (25 35 1) simpleGrading (0.15 1 1)
    hex (8 9 10 11 12 13 14 15)   (75 35 1) simpleGrading (1 1 1)
    hex (16 17 18 19 20 21 22 23) (25 35 1) simpleGrading (6.66666 1 1)
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
            (1 5 4 0)     // Suelo del cubo 1
            (16 17 21 20) // Suelo del cubo 3
        );
    }
    inlet_hidrofilic // Suelo del cubo
    {
        type patch;
        faces
        (
            (8 9 13 12)   // Suelo del cubo 2
            (11 10 14 15) // Techo del cubo 2

        );
    }
    atmosphere // Techo del cubo
    {
        type patch;
        faces
        (
            (3 7 6 2)     // Techo del cubo 1
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

### Archivo SetFields, incialización de campos en fluidos
Este archivo nos dirá el estado y la geometría en la cual inician nuestras gotas, es decir velocidad, posición, presión inicial.
Para nuestra simulación tomaremos como condición inicial de las gotas una velocidad = 0 y también tomaremos semi círculos de las gotas apoyados en el suelo (como se puede apreciar en la figura).

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
        p1 (0.01 0 0);
        p2 (0.01 0 0.02);
        radius 0.002;//CHANGE

        fieldValues
        (
            volScalarFieldValue alpha.water 1
            volVectorFieldValue U (0 0 0)
        );
    }
    cylinderToCell
    {
        p1 (0.02 0 0);
        p2 (0.02 0 0.02);
        radius 0.002;//CHANGE

        fieldValues
        (
            volScalarFieldValue alpha.water 1
            volVectorFieldValue U (0 0 0)
        );
    }
);


// ************************************************************************* //
```

### Condiciones de contorno
Las condiciones de contorno de acuerdo a lo planteado anteriormente serán:

Para la velocidad:
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
    class       volVectorField;
    location    "0";
    object      U;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 1 -1 0 0 0 0];

internalField   uniform (0 0 0);

boundaryField
{
    inlet_hidrofobic
    {
        type            noSlip;
        /*
        type            pressureInletOutletVelocity;
        value           uniform (0 0 0);
        */
    }

    inlet_hidrofilic
    {
        type            noSlip;
        /*
        type            pressureInletOutletVelocity;
        value           uniform (0 0 0);
        */
    }

    atmosphere
    {
        type            noSlip;
    }

    walls
    {
        type            pressureInletOutletVelocity;
        value           uniform (0 0 0);
    }

    defaultFaces
    {
        type            empty;
    }
}


// ************************************************************************* //
```

Para la presión dinámica:
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
    class       volScalarField;
    object      p_rgh;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [1 -1 -2 0 0 0 0];

internalField   uniform 0;

boundaryField
{
    inlet_hidrofobic
    {
        type            fixedFluxPressure;
    }

    inlet_hidrofilic
    {
        type            fixedFluxPressure;
    }


    atmosphere
    {
        type            fixedFluxPressure;

    }

    walls
    {
        type            fixedValue;
        value           uniform 0;
    }

    defaultFaces
    {
        type            empty;
    }
}


// ************************************************************************* //
```

Para los ángulos de contacto
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
    class       volScalarField;
    object      alpha.water;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 0 0 0 0 0 0];

internalField   uniform 0;

boundaryField
{
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
        theta0          45;//CHANGE
        limit           gradient;
        value           uniform 0;
    }

    atmosphere
    {
        type            constantAlphaContactAngle;
        theta0          135;
        limit           gradient;
        value           uniform 0;
        /*
        type            zeroGradient;
        */
    }

    walls
    {
        type           constantAlphaContactAngle;
        theta0         90;
        limit          gradient;
        value          uniform 0;
    }

    defaultFaces
    {
        type            empty;
    }
}


// ************************************************************************* //
```


---
# Script de Bash

Este script será el encargado de realizar la ejecución de comandos necesarios para la ejecusión de los casos de open foam. Todo esto será hecho a partir de un caso original o caso **madre** como le hemos puesto. Este caso será copiado y pegado multiples veces, cambiando ciertos parámetros de ciertos archivos específicos para mover las variables deseadas.

```bash
#!/bin/bash

# Variables iniciales
CARPETA_ORIGINAL="mother-case"
ARCHIVO_A_MODIFICAR_RADIO="system/setFieldsDict"
ARCHIVO_A_MODIFICAR_ANGULO="0.orig/alpha.water.orig"
RADIO_INICIAL=0.0025
RADIO_FINAL=0.0015
ANGULO_HIDROFILICO_INICIAL=50
ANGULO_HIDROFILICO_FINAL=90
PASO_ANGULAR=5 # Esto es para ir sumando al ángulo inicial
PASO=-0.0001 # Esto es para ir restandole al radio inicial
PALABRA_BASE="radius 0.002"  # Para usar en la búsqueda y reemplazo
PALABRA_BASE_ANGULO="45"  # Para usar en la búsqueda y reemplazo

for ANGULO_NUEVO in $(seq $ANGULO_HIDROFILICO_INICIAL $PASO_ANGULAR $ANGULO_HIDROFILICO_FINAL); do
    for RADIO_NUEVO in $(seq $RADIO_INICIAL $PASO $RADIO_FINAL); do
        
        RADIO_CON_PUNTO=$(printf "%.4f" "$RADIO_NUEVO" | sed 's/,/./g')
        NUEVA_CARPETA="${CARPETA_ORIGINAL}-radius-${RADIO_CON_PUNTO}-angle-${ANGULO_NUEVO}"

        if [ -d "$CARPETA_ORIGINAL" ]; then
            cp -r "$CARPETA_ORIGINAL" "$NUEVA_CARPETA"
            echo "Carpeta copiada como: $NUEVA_CARPETA"
        else
            echo "La carpeta original '$CARPETA_ORIGINAL' no existe."
            exit 1
        fi

        cd "$NUEVA_CARPETA" || { echo "No se pudo acceder a la carpeta $NUEVA_CARPETA"; exit 1; }

        if [ -f "$ARCHIVO_A_MODIFICAR_RADIO" ]; then
            PALABRA_A_BUSCAR="$PALABRA_BASE"
            PALABRA_NUEVA="radius ${RADIO_CON_PUNTO}"
            sed -i "s/$PALABRA_A_BUSCAR/$PALABRA_NUEVA/g" "$ARCHIVO_A_MODIFICAR_RADIO"
            echo "Se modificó el archivo $ARCHIVO_A_MODIFICAR_RADIO reemplazando '$PALABRA_A_BUSCAR' por '$PALABRA_NUEVA'."
        else
            echo "El archivo '$ARCHIVO_A_MODIFICAR_RADIO' no existe en la carpeta copiada."
            exit 1
        fi
        
        if [ -f "$ARCHIVO_A_MODIFICAR_ANGULO" ]; then
            PALABRA_NUEVA_ANGULO="${ANGULO_NUEVO}"
            PALABRA_VACIA=""
            if grep -q "$PALABRA_BASE_ANGULO" "$ARCHIVO_A_MODIFICAR_ANGULO"; then
                sed -i "s/$PALABRA_BASE_ANGULO/$PALABRA_NUEVA_ANGULO/g" "$ARCHIVO_A_MODIFICAR_ANGULO"
                sed -i "s/CHANGE/$PALABRA_VACIA/g" "$ARCHIVO_A_MODIFICAR_ANGULO"               
                echo "Se modificó el archivo $ARCHIVO_A_MODIFICAR_ANGULO reemplazando '$PALABRA_BASE_ANGULO' por '$PALABRA_NUEVA_ANGULO'."
            else
                echo "No se encontró '$PALABRA_BASE_ANGULO' en $ARCHIVO_A_MODIFICAR_ANGULO. Reemplazo omitido."
            fi
        else
            echo "El archivo '$ARCHIVO_A_MODIFICAR_ANGULO' no existe en la carpeta copiada."
            exit 1
        fi

        eval "source /opt/openfoam9/etc/bashrc"
        eval "./Allrun"

        cd ..

        echo "Iteración con radio $RADIO_CON_PUNTO y ángulo $ANGULO_NUEVO completada."
    done
done

echo "Script completado con éxito."

```

---
# Conclusiones
Finalmente podemos concluir que combinar la potencia de la automatización de linux junto con la potencia de cálculo de open foam, nos puede proporcionar una base muy sólida para la generación de datos a partir de simulaciones. Mas aún si combinamos el poder de la paralelización que es capaz de realizar Open Foam, junto con la automatización serial que es capaz de producir linux con sus script de bash.

## Material referente a este post

- [Video de una de las simulaciones](https://youtu.be/TKTCwr6PEuU)
- [Link al caso madre y script de bash para generar todos los casos](https://drive.google.com/drive/folders/1jvRHyZLux1YNsMk3xUBB3CeE2unBiwCi?usp=sharing)

