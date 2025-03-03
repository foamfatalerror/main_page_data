# Introducción 
## ¿De que se trata?

Las funciones de postprocesamiento en OpenFOAM son una herramienta poderosísima que nos permite lograr calcular una serie de cálculos de campos, magnitudes, variables o coeficientes en regiones o puntos de interes (puede ser todo el dominio también).

Estas funciones se suelen llamar de postprocesamiento porque derivan del cálculo principal y no son piezas centrales del cálculo, si no que son agregados que nos aportan información suplementaria el estudio que estamos intentando hacer.

## Consideraciones importantes

Para este post, estaremos utilizando la versión de **OpenFOAM 2406** y utilizaremos un solver particular llamado `icoReactingMultiphaseInterFoam`. Si bien en otras versiones de OpenFOAM el lugar de la función cambia, el concepto de fondo es el mismo con lo cual la información que exponemos aquí puede serte de utilidad igualmente.

# ¿Donde se encuentran?

Las funciones prefabricadas que trae open foam las podemos encontrar dentro de la carpeta:
```bash
$FOAM_SRC/functionObjects/
```
Donde `$FOAM_SRC` es la dirección donde tenemos instalado OpenFOAM en nuestra PC. En mi caso la dirección es:
```bash
/usr/lib/openfoam/openfoam2406/src
```
Si lanzamos un ls en `$FOAM_SRC/functionObjects/` veremos:
```bash
/usr/lib/openfoam/openfoam2406/src/functionObjects
├── Allwmake
├── doc
├── field
├── forces
├── graphics
├── initialisation
├── lagrangian
├── phaseSystems
├── randomProcesses
├── solvers
└── utilities
```
En nuestro caso utilizaremos las funciones dentro de field, que son las asociadas a los campos. Dentro de esta carpeta podremos ver:
```bash
/usr/lib/openfoam/openfoam2406/src/functionObjects/field
├── add
├── age
├── AMIWeights
├── binField
├── blendingFactor
├── cellDecomposer
├── columnAverage
├── comfort
├── components
├── continuityError
├── CourantNo
├── Curle
├── ddt
├── ddt2
├── derivedFields
├── DESModelRegions
├── div
├── DMD
├── doc
├── enstrophy
├── expressions
├── externalCoupled
├── extractEulerianParticles
├── fieldAverage
├── fieldCoordinateSystemTransform
├── fieldExpression
├── fieldExtents
├── fieldMinMax
├── fieldsExpression
├── fieldValues
├── flowType
├── flux
├── fluxSummary
├── grad
├── heatTransferCoeff
├── histogram
├── interfaceHeight
├── Lambda2
├── LambVector
├── limitFields
├── lnInclude
├── log
├── MachNo
├── mag
├── magSqr
├── Make
├── mapFields
├── momentum
├── momentumError
├── multiFieldValue
├── multiply
├── nearWallFields
├── norm
├── particleDistribution
├── PecletNo
├── pow
├── pressure
├── processorField
├── proudmanAcousticPower
├── Q
├── randomise
├── reactionSensitivityAnalysis
├── readFields
├── reference
├── regionSizeDistribution
├── resolutionIndex
├── setFlow
├── stabilityBlendingFactor
├── streamFunction
├── streamLine
├── subtract
├── surfaceDistance
├── surfaceInterpolate
├── turbulenceFields
├── valueAverage
├── vorticity
├── wallBoundedStreamLine
├── wallHeatFlux
├── wallShearStress
├── writeCellCentres
├── writeCellVolumes
├── XiReactionRate
├── yPlus
└── zeroGradient
```
Estas son todas las funciones que pueden calcular campos en nuestra simulación. En nuestro caso utilizaremos `wallHeatFlux` que se refiere al flujo de calor que experimentan los bordes de condición **Wall**. Esto es sumamente importante ya que si nuestra condición de borde **no es wall** la función prefabricada **no podrá** calcular el flujo de calor en el borde.

# Caso ejemplo
Para poder poner a prueba las funciones de postprocesamiento, utilizaremos un caso de ejemplo de una gota que se encuentra sobre una superficie caliente (que la hace hervir) mientras que el techo es una superficie fria (que hace condensar el vapor). Esto provocará una transferencia de calor entre suelo y techo que podremos ver con nuestra función `wallHeatFlux`.

Por otro lado esta simulación de ejemplo tendrá las siguientes consideraciones:
- Solver: `icoReactingMultiphaseInterFoam`
- Tipo: 2-D (tendremos un eje z que no utilizaremos).

Esquema del dominio de estudio:
![imagen_situacion_fisica](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202503-introduccion-a-funciones-post-procesamiento/dibujo-dominio-gota-hirviendo.png)
# ¿Como se utilizan?
Las funciones de **postprocesamiento** (al menos en la versión de OpenFOAM 2406) se escriben dentro de nuestro carpeta del caso en el archivo `system/controlDict`.

Nuestro archivo `controlDict` con la funcion agregada de `wallHeatFlux` se verá de la siguiente manera:
```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2406                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      controlDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

application     icoReactingMultiphaseInterFoam;
startFrom       latestTime;
startTime       0;
stopAt          endTime;
endTime         5;
deltaT          1e-3;
writeControl    adjustable;
writeInterval   0.0025;
purgeWrite      0;
writeFormat     ascii;
writePrecision  6;
writeCompression off;
timeFormat      general;
timePrecision   6;
runTimeModifiable yes;
adjustTimeStep  yes;
maxDeltaT       1e-1;
maxCo           0.5;
maxAlphaCo      1.5;
maxAlphaDdt     1;

// Aqui es donde sucede lo importante
functions
{
	wallHeatFluxFunction
	{
	    type            wallHeatFlux;
	    libs            ("libfieldFunctionObjects.so");
	    patches         (top bottom);
	    writeControl    adjustableRunTime;
	    writeInterval   0.0025;
	}
}
// ************************************************************************* //
```

En donde el apartado de funciones nos permitirá escribir las funciones de aquello que necesitamos.

# Resultados

Podemos ver los resultados de esta simulación en el siguiente video:
[Link al video](https://youtu.be/7t2upXiz1Y0)

En donde veremos los campos principales de la simulación y por último veremos el flujo de calor en las paredes que es lo que nos interesa.

Tambien te dejamos el código del caso para que puedas testearlo:
[Repositorio con el código](https://drive.google.com/drive/folders/1XiVVsrwCns3iJqlZGtSab2i8TvCK7KE6?usp=sharing)




