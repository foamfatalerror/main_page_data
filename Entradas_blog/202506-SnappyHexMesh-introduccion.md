# Introducción

## ¿Que es Snappy Hex Mesh?

Snappy Hex Mesh es una herramienta de software libre acoplada a **OpenFOAM** que es capaz de mallar geometrías complejas. Si ya pasaste por nuestro post de **blockMesh** o si conoces la herramienta, sabras que es muy difícil poder mallar una geometría con una pequeña complejidad, ni que hablar de una muy complicada como solemos necesitar en el día a día en CFD. Ejemplos de geometrías complejas hay miles pero por mencionar algunas podrían ser turbinas de avión, vehículos,  generadores eólicos, (etc). Con BlockMesh sería simplemente imposible.

Es por esto que surge la necesidad de una herramienta que nos permita mallar este tipo de geometrías y que ademas nos permita configurar nuestro mallado de tal manera que podamos refinar en zonas importantes, poner mayor cantidad de capas de refinado en otras, adaptar el mallado a las curvas del dominio, etc.

Todo esto es lo que podemos lograr utilizando la herramienta **SnappyHexMesh** que se presenta como una solución integral al problema del mallado y también al problema de la personalización del mismo. Dándonos opciones incluso como paralelizar el mallado para hacerlo mas veloz (para ciertas geometrías, no es trivial calcular la malla, podría llevar horas).

## ¿Que vamos a hacer en este tutorial?

En este tutorial vamos a explorar los fundamentos de **snappyHexMesh** y algunas de sus opciones, también les mostraré de que manera podemos mallar en paralelo (es decir utilizando varios núcleos del procesador).

Vamos a tomar una geometría no trivial (en nuestro caso el modelo 3D de una botella de cerveza):
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/render-beer.png)

La idea es lograr un mallado que se adapte a esta geometría mas bien curva y mallar su interior correctamente, con un mallado mas fino en sus paredes.
## Links importantes y fuentes (Completar):

1. [Link al repositorio del caso](https://drive.google.com/drive/folders/1bgg6BXNNu_ejHLvbaQziGcOb0CXGBYZC?usp=sharing)
2. [Link a presentación con explicaciones detalladas de SnappyHexMesh](https://www.wolfdynamics.com/wiki/meshing_OF_SHM.pdf)
## Consideraciones importantes:

1. Estamos utilizando OpenFOAM en su versión .com
2. Es la versión v2412
3. Si no leíste nuestro post sobre **blockMesh** o no conoces blockMesh, te invitamos a que lo aprendas primero y luego vuelvas a este post, ya que es fundamental tener ese conocimiento primero.

---
# Como lo implementamos

La manera de implementar snappyHexMesh en nuestros casos es a través de un diccionario llamado **snappyHexMeshDict**  (si, super original el nombre) y generalmente va acompañado de otro diccionario llamado **surfaceFeatureExtractDict** . 
## ¿Donde van los diccionarios y que archivos necesitamos para comenzar?

Estos diccionarios van colocados dentro de la carpeta `system` :

```
├── 0.orig
│   ├── air.gas
│   ├── alpha.gas
│   ├── alpha.liquid
│   ├── p_rgh
│   ├── T
│   ├── U
│   └── vapour.gas
├── Allclean
├── Allmesh
├── Allmesh_serial
├── Allrun
├── constant
│   ├── g
│   ├── phaseProperties
│   ├── thermophysicalProperties.gas
│   ├── thermophysicalProperties.liquid
│   ├── triSurface
│   │   └── beer.stl <------------------------ Modelo 3D en STL a utilizar
│   └── turbulenceProperties
├── Models
│   └── beer.stl <---------------------------- Modelo 3D en STL a utilizar
└── system
    ├── blockMeshDict
    ├── controlDict
    ├── decomposeParDict
    ├── fvSchemes
    ├── fvSolution
    ├── setFieldsDict
    ├── snappyHexMeshDict <------------------- Diccionario de Snappy
    └── surfaceFeatureExtractDict <----------- Diccionario complementario

```

Ademas de estos dos diccionarios, lo otro que necesitaremos es tener nuestros STL que capturen la geometría de lo que intentaremos mallar. En nuestro caso lo tendremos repetido en dos carpetas. Primero lo pondremos en una carpeta llamada `Models` que está a la altura de las carpetas `system`, `constant` y `0.orig`. En esta carpeta tendremos una copia de seguridad de nuestro modelo 3D por cualquier necesidad.

Luego también estará dentro de la carpeta `constant/triSurface` allí dentro será en donde OpenFOAM lo buscará a la hora de leer las geometrías.

Con estos diccionarios en su lugar y los modelos 3D también, tenemos todo para empezar a configurar nuestro `snappyHexMeshDict` y `surfaceFeatureExtractDict`
## ¿De que manera los configuramos?

Primero que nada tenemos que entender que hará cada diccionario. 
1. `surfaceFeatureExtractDict`: Se encargará de extraer todas las curvas que sean demasiado cerradas o con ángulos superando un umbral (por ejemplo 30°). Esto asegura que las esquinas "filosas" o con ángulos agudos, sean capturadas para luego tener especial cuidado en esas zonas.
2. `snappyHexMeshDict`: Se encargará de mallar literalmente sobre el dominio que deseamos, tomando parámetros que le dejará `surfaceFeatureExtractDict` en una carpeta para que pueda ser utilizado.

Una vez entendido esto, comenzamos a configurarlos en el orden que los utilizaremos.

### `system/surfaceFeatureExtractDict`

Este diccionario es el mas sencillo de ambos ya que tiene pocos parámetros.
```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2412                                  |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      surfaceFeatureExtractDict;
}

beer.stl // Nombre del STL del que vamos a extraer las curvas
{
    // Definimos la manera de extraer los datos (extractFromFile o extractFromSurface)
    extractionMethod    extractFromSurface;

    extractFromSurfaceCoeffs
    {
        // Marca las aristas cuyas normales de superficie adyacentes 
        // tienen un ángulo menor al que le seteemos. 
        // Es decir, lee los ángulos "filosos".
        // El ángulo tomado es 180 - anguloIndicado.
        // - 0  : No seleccionamos ningúna curva del modelo 3D
        // - 180: Seleccionamos todas las curvas del modelo 3D
        includedAngle   150; // Es decir que si es <30° lo tomaremos
    }

    // Opciones de escritura
    // Escribe en forma de .obj las curvas que fueron seleccionadas
    // de la geometría para poder verlas en paraview por ejemplo.
        writeObj                yes;
}
```

### `system/snappyHexMeshDict`

Este diccionario lo separaremos en tres partes para poder abarcarlo mejor. Se estructura de la siguiente manera:

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
    object      snappyHexMeshDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

castellatedMesh true;
snap            true;
addLayers       true;


geometry
{
    cuerpo //Nombre del modelo 3D
    {
        type triSurfaceMesh;
        file "beer.stl";
    }

};


castellatedMeshControls
{...} 
snapControls
{...}
addLayersControls
{...}
meshQualityControls
{...}

mergeTolerance 1e-6;

// ************************************************************************* //
```

En donde:
```cpp
castellatedMesh true;
snap            true;
addLayers       true;
```

Nos dice que cosas haremos con la malla. 

1. `castellatedMesh`: Es casi obligatoria ya que es la malla base con algunos refinados en ciertas zonas aunque todas sus celdas son hexahedricas, es decir que el resultado salido de tener solo en true esta opción, es una malla "pixelada" sin seguir curvas o bordes suaves.
2. `snap`: Es la ajustará algunas celdas a las curvas de nuestra geometría, permitiendo bordes suaves en zonas que lo requieran, especialmente en bordes de la geometría.
3. `addLayers`: Es la que agregará capas de refinamiento extra en zonas deseadas, esto nos permitirá darle especial tratamiento a zonas mas "problemáticas" o donde deseemos mirar. Se usa para **agregar capas prismáticas** sobre las superficies de la geometría. Estas capas son muy útiles para capturar correctamente fenómenos de capa límite en simulaciones de flujo.

Luego:
```cpp
geometry
{
    cuerpo //Nombre del modelo 3D
    {
        type triSurfaceMesh;
        file "beer.stl";
    }
};
```

Es en donde importaremos nuestros modelos 3D en formato STL (todos los que estén dentro de triSurface). En este caso solo tenemos un modelo llamado `beer.stl`.

#### `castellatedMeshControls`

```cpp
castellatedMeshControls
{
	// Esto es el máximo de celdas "locales" permitidas para nuestro mallado
    maxLocalCells 1000000; 
	
	// Mientras que este es nuestro máximo global con todas las opciones aplicadas (castellatedMesh, snap y addLayers)
    maxGlobalCells 2000000;
	
	// Esto es para proponer un mínimo de celdas que necesitemos/queramos
    minRefinementCells 0;
	
	// Define el "desequilibrio máximo permitido" entre la cantidad de trabajo (celdas a generar) que se asigna a los distintos procesadores. Esto solo nos importa si mallamos en paralelo.
    maxLoadUnbalance 0.10;

	// Cantidad mínima de celdas entre zonas de diferente nivel de refinamiento en el mallado.
    nCellsBetweenLevels 3;

	// Caracteristicas que leeremos de lo extraido con surfaceFeatureExtractDict. Podemos definir un nivel 2 (o superiores) de refinamiento en las curvas mas "peligrosas" o "angulosas" según nuestro criterio utilizado en surfaceFeatureExtractDict.
    features
    (
        {
            file "beer.eMesh";
            level 2;
        }
    );

	// Definimos que vamos a mallar mas fino en algunas curvas mas intensas de la superficie que declaremos, en nuestro caso es nuestro mismo cuerpo, pero podría ser un segundo stl creado solo para refinar mas intensamente en ese punto. 
    refinementSurfaces
    {
        cuerpo
        {
            // En zonas que cumplan el criterio pasamos de un mallado nivel 1 a 2
            level           (1 2);
			// (nivelMáximoCurvatura nivelMínimoCurvatura curvaturaAPartirDeLaCualActivamos nivelRefinamientoMínimo);
            curvatureLevel (10 0 10 -1);
        }
    }

	// Con este detectamos zonas con ángulos menores a 60° y refinamos mas en esas zonas.
    resolveFeatureAngle 60;

	// Esta es una opción que se utiliza para refinar de manera volumétrica. A diferencia de refinementSurfaces que lo hará por caras del STL, refinementRegions lo hará por todo el volumen del STL que le carguemos (puede ser distinto al que utilizamos)
    refinementRegions
    {
    }

    // Esta es la locación del punto sobre el cual definimos el dominio. Solo hay dos opciones o estamos dentro del objeto o estamos fuera. En nuestro caso estaremos dentro de la botella, ese será nuestro dominio
    locationInMesh (0 0 0.1);

	// Esta opción es más avanzada y controla si `snappyHexMesh` permite **caras internas que separan zonas (zones)** aunque no estén asociadas a una superficie STL.

// - `true`: Se permiten estas caras, útil por ejemplo para definir zonas internas como poros, filtros, o simplemente separar zonas sin una geometría STL que las defina.
    
// - `false`: Se eliminan esas caras si no hay superficie física que las justifique.
    allowFreeStandingZoneFaces true;

}

```

Si ejecutamos solo con `castellatedMesh` en `true` y `snap` en `false` obtendremos una malla llena de bloques hexagonales y una malla 100% ortogonal, es decir que no se curvará para adaptarse a la forma de la superficie 3D
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/beer-without-snap.png)

#### `snapControls`

Esta opción es un poco mas sencilla ya que solo estamos tratando con la adaptación de la malla a las curvas de la geometría. Lo que hará el algoritmo es literalmente "mover" los nodos de las celdas cercanas al exterior de la geometria o a sus curvas, para adaptarla lo mejor posible.

```cpp
snapControls
{
	// Número de iteraciones de suavizado de los puntos de la malla en la superficie luego del snapping.
    nSmoothPatch 3;
    
    // significa que si los puntos están a menos de 2 veces el tamaño de celda de la geometría, se considera que están suficientemente cerca y no se siguen moviendo.
    tolerance 2; 

	// Cantidad de iteraciones para resolver el sistema que mueve los puntos hacia la geometría. Valores comunes: 20–100.
    nSolveIter 50;
    
    // Número de "iteraciones de relajación" que se aplican a los desplazamientos antes de mover los puntos. Es basicamente para evitar divergencias en estos cálculos.
    nRelaxIter 5;//8;

	// Cantidad de iteraciones dedicadas a ajustar los puntos específicamente a las features (aristas agudas de surfaceFeatureExtractDict)
    nFeatureSnapIter 15;

	// Permite a `snappyHexMesh` detectar automáticamente features a partir de la geometría STL sin que estén explícitamente definidas en otro archivo (como `.eMesh`).
    implicitFeatureSnap true;

	// Permite hacer snapping a features definidas explícitamente, usualmente mediante un archivo `.eMesh`. El que obtuvimos de surfaceFeatureExtractDict
    explicitFeatureSnap true;
}
```

![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/beer-mesh.png)

![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/beer-front.png)

#### `addLayersControls`

```cpp
addLayersControls
{
	// Este parámetro define si los tamaños de las capas deben interpretarse relativamente al tamaño local de celda
    relativeSizes true;

	// Acá se define cuantas capas de celdas se agregan sobre una superficie específica.
    cuerpo{
        nSurfaceLayers 2;
    }

	// Este parámetro controla el crecimiento geométrico entre capas sucesivas.
    expansionRatio 1.2;
	
	// Define el espesor de la primera capa, ya sea en unidades relativas o absolutas.
    firstLayerThickness 0.01;
}
```


![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/beer-cut.png)

![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202506-SnappyHexMesh-introduccion/beer-cut-lateral.png)

#### `meshQualityControls`

Este subdiccionario es muy particular, ya que solo es declarado para realizar un "chequeo" de calidad de la malla. Estableceremos criterios que, de no cumplirse, snappy nos avisará o hará incluso fallar el mallado para evitar un producto defectuoso según nuestros criterios.

```cpp
meshQualityControls
{
	// Límite máximo del ángulo de no ortogonalidad entre el vector de cara y la línea entre centros de celdas.
    maxNonOrtho 65;

	// Controlan la distorsión o sesgo entre el centro de la celda y el centro de la cara.
    maxBoundarySkewness 20; // para bordes de la malla
    maxInternalSkewness 4; // para interiores de la malla

	// Controla el ángulo máximo permitido en celdas cóncavas.
    maxConcave 80;

	// Volumen mínimo permitido para una celda.
    minVol 1.00E-13;

	
    // Controla la calidad mínima de los tetraedros internos usados en verificación de calidad.
    minTetQuality 1e-9;

	// Área mínima permitida de una cara.
    minArea -1;

	// Controla qué tan “torcidas” pueden estar las caras respecto al alineamiento entre centros de celdas.
    minTwist 0.02;
    
    // Verifica la "invertibilidad del tensor de transformación" de la celda. Si es menor a 0, la celda está invertida.
    minDeterminant 0.001;

	// Relación entre el área proyectada de la cara y su área real.
    minFaceWeight 0.05;

	// Relación entre el volumen mínimo de celdas vecinas.
    minVolRatio 0.01;

	// Desactiva el chequeo de **twist** en caras triangulares.
    minTriangleTwist -1;

	// Controla qué tan planas deben ser las caras.
    minFlatness 0.5;

	// Cuántas veces se permite suavizar la malla para mejorar calidad antes de abortar.
    nSmoothScale 4;
    
	// Factor de reducción de errores que se busca en cada iteración de mejora. Es decir que por iteración mejore un 25%
    errorReduction 0.75;
}
```
## ¿ Como hacemos para mallar en serie?

Los comandos que deberemos lanzar son:

```bash
blockMesh

surfaceFeatureExtract

snappyHexMesh -overwrite
```

## ¿Como hacemos para mallar en paralelo (con varios nucleos del procesador, paralelizado)?

Para lograr esto primero deberemos tener bien seteado otro diccionario llamado `system/decomposeParDict`

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
    object      decomposeParDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

numberOfSubdomains 6; // cantidad de trabajo en paralelo.

method          simple;

coeffs
{
	// Partimos el dominio en "x" en 2 partes iguales y en "y" en tres
    n           (2 3 1);
}

// ************************************************************************* //
```

Con este diccionario haremos un mallado paralelo utilizando 6 cores o nucleos.

Luego los comandos que deberemos lanzar son (archivo Allmesh):
```bash
#!/bin/sh
cd "${0%/*}" || exit                                # Run from this directory
. ${WM_PROJECT_DIR:?}/bin/tools/RunFunctions        # Tutorial run functions
#------------------------------------------------------------------------------

# canCompile || exit 0    # Dynamic code

restore0Dir

touch foam.foam

runApplication blockMesh

runApplication surfaceFeatureExtract

## Parallel Mesh
runApplication decomposePar 
runParallel    snappyHexMesh -overwrite 
runApplication reconstructParMesh -constant 
```


---
# Conclusiones

**SnappyHexMesh** puede ser desafiante para un inicio por su cantidad de opciones y conceptos geométricos involucrados que puede asustarnos en un inicio, sin embargo es altamente recomendado para mallados altamente personalizables y a menudo es a la opción a donde vamos para mallar geometrías increiblemente complejas.

Como un consejo para aquellos que estén recién comenzando con snappy, es mejor concentrarse en el subdiccionario `castellatedMeshControls` y dejar todo en false:

```cpp
castellatedMesh true;
snap            false;
addLayers       false;
```

Esto nos dará menos cosas de que preocuparnos y aun así obtendremos un mallado aproximado bueno, aunque quizá sin curvas. Sin embargo para comenzar será mas que suficiente.