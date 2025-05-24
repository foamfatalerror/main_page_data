# Introducción

El **postprocesamiento** es una etapa esencial en toda simulación numérica, ya que permite interpretar, analizar y visualizar los resultados obtenidos de forma clara y eficiente. Ademas de calcular algunas métricas o campos que no vienen precalculadas por defecto en la simulación.

En este post en particular vamos a explorar como aplicar con `coded functions` algunos cálculos de metricas no triviales y que involucran el cálculo de muchos campos dentro de nuestra simulación.
## Consideraciones iniciales

- Todas las pruebas y todo el tutorial en sí, estarán hechos en **OpenFOAM 2406** en su versión .com
- Utilizaremos **interFoam** para estudiar estas funciones, sin embargo funciona cualquier sea el solver utilizado. Este solver contempla flujos multifásicos, incompresibles e isotérmicos.
- Es importante conocer los conceptos de **campos escalares**, **campos vectoriales**, **integrales de volumen** y operaciones entre campos debido a que las funciones de postprocesamiento tienen esta capacidad de cálculo numérico.

## Que vamos a hacer en este tutorial?

Vamos a calcular cinco magnitudes interesantes de una simulación genérica de una gota grande de agua cayendo por un túnel de sección cuadrada que está sometido a vientos ascendentes.
Esto podemos apreciarlo en la siguiente figura:

![Geometria](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202505-funciones-post-procesamiento-intermedio/geometria.png)

Inicialmente tendremos una gota esférica en el centro del volumen la cual caerá. Agregando la fuerza gravitatoria, la gotá sentirá una fuerza hacia la "izquierda" de la imagen.
![imagen_agua](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202505-funciones-post-procesamiento-intermedio/agua.png)

Las métricas que calcularemos para nuestro caso serán:

1. Centro de masa de la Gota.
  $$
  \vec{X}_{cm}= \frac{\int_V\rho\cdot x \cdot dV}{\int_V\rho \cdot dV}
  $$
2. Área de interfaz líquido-gas (superficie libre)
  $$
  A \approx \int_V |\nabla \alpha|dV
  $$

3. Fuerzas que siente la gota, tanto por tensión superficial como por presión.
$$
F_g = F_\sigma + F_p
$$
$$
F_\sigma = \sum_{n=1}^{celdas} -\nabla \left( \frac{\nabla \alpha}{|\nabla \alpha|} \right)_{faces} \cdot \sigma \cdot S 
$$
$$
F_p = \sum_{n=1}^{celdas} P_{rgh} \cdot S \cdot  \hat{n}
$$
	En donde $\sigma$ es tensión superficial, $\alpha$ es concentración de líquido y $S$ es superficie.
4. Energía cinética total de la fase líquida:
$$
E_k = \frac{1}{2}\int_V \rho \alpha |\vec{U}|^2 dV
$$
	En donde $\rho$ es densidad, $U$ es velocidad y $dV$ es diferencial de volumen.
5. Energía cinética total de la fase gaseosa:
$$
E_k = \frac{1}{2}\int_V \rho \alpha |\vec{U}|^2 dV
$$

---
## Material adicional
- [Link al Caso](https://drive.google.com/open?id=15YahWZzIhp3QrqNpxyK5NIBrwZdUK_8V&usp=drive_copy)
- 
---
# ¿Como se utiliza?

Una vez que ya sabemos que métricas deseamos calcular, ahora tenemos que ir a nuestro caso y comenzar a escribir las `coded functions` que calcularán están métricas y campos que deseamos conocer.

Esto se escribe generalmente en nuestro archivo llamado `system/controlDict` en donde tenemos información sobre pasos temporales, numero courant, opciones de la simulación y opciones mas generales.

```
├── 0.orig
│   ├── alpha.water
│   ├── p_rgh
│   └── U
├── Allclean
├── Allrun
├── constant
│   ├── g
│   ├── transportProperties
│   └── turbulenceProperties
└── system
    ├── blockMeshDict
    ├── controlDict  <---- Aqui escribiremos nuestras funciones
    ├── fvSchemes
    ├── fvSolution
    └── setFieldsDict
```

A su vez dentro de este de file, nos encontraremos con algo muy similar a esto:

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
    object      controlDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

application     interFoam;

startFrom       startTime;

startTime       0;

stopAt          endTime;

endTime         1;

deltaT          1e-3;

writeControl    adjustable;

writeInterval   0.01;

purgeWrite      0;

writeFormat     ascii;

writePrecision  6;

writeCompression off;

timeFormat      general;

timePrecision   6;

runTimeModifiable yes;

adjustTimeStep  yes;

maxCo           1;

maxAlphaCo      1;

maxDeltaT       1;

```

Justo debajo de la última linea es donde comenzará el desarrollo. Tendremos que abrir un diccionario llamado `functions` y allí dentro escribiremos las funciones:
```cpp
functions
{
    centerOfMass
    {...}

	interfaceArea
	{...}

	dropForce
	{...}

	kineticEnergyOfWater
	{...}

	kineticEnergyOfWind
	{...}
}
```

## Algunos conceptos y tipos de variables importantes
- `volVectorField`: Cuando declaramos este tipo de variables, nos referimos a un objeto que representa un campo vectorial, con lo cual va a ser la colección de campos vectoriales de cada celda. Por ejemplo la velocidad U, en donde cada celda tendrá una lista de tres numeros que representarán su velocidad (Ux, Uy, Uz).
- `volScalarField`: Cuando declaramos este tipo de variables, nos referimos a un objeto que representa un campo escalar, con lo cual va a ser la colección de campos escalares de cada celda. Por ejemplo la presión, en donde cada celda tendrá un escalar que representará su presión.
-  `surfaceVectorField`: Representa campo vectorial, pero que está aplicado sobre las caras de la celda y no en su centro como lo es un `volVectorField`.
- `surfaceScalarField`: Representa campo escalar, pero que está aplicado sobre las caras de la celda y no en su centro como lo es un `volScalarField`.
- `mesh()`: Representa al objeto malla, del cual sacamos todo lo relacionado a la geometria, ej: superficies, volúmenes, direcciones, etc.
- `fvc::grad(campo)`: Gradiente de un campo
- `fvc::div(campo)`: Divergencia de un campo.
- `fvc::interpolate(campo)`: Lleva un campo que está centrado en una celda, hacia las caras. Esto lo hace interpolando los valores.
- `fvc::snGrad(campo)`: Calcula el gradiente del campo en la superficie de la celda y no en su centro. 
- `mesh().magSf()`: Area de las caras de las celdas.
- `mesh().V()`: Volumen de las celdas
## Función de centro de masa.

```cpp
    centerOfMass
    {
        type            coded;
        libs            ("libutilityFunctionObjects.so");
        name            centerOfMass;

        codeExecute
        #{
        	// Llamamos al campo escalar "alpha" que es el porcentaje de agua que tenemos en la celda
            const volScalarField& alpha = mesh().lookupObject<volScalarField>("alpha.water");
	
			// En esta linea estamos haciendo la integral del centro de masa que tenemos en la sección teorica "que haremos" en donde:
			// sum() es la función que realiza la integración numérica celda por celda.
			// mesh().C() trae la posición tridimensional de cada celda, es decir: (x,y,z)
			// .value() de cualquier calculo/campo/magnitud lo que hace es quitarle las unidades y tomarlo solo como un número.
            vector com = (sum(alpha * mesh().C()).value()) / sum(alpha).value();
			
			// En este paso lanzamos al log que es lo que está haciendo la simulación en este momento.
            Info<< "Time: " << time().timeName() << " | Center of Mass: " << com << nl;
        
            // Crear carpeta si no existe
            fileName dirName = "postProcessing/centerOfMass";
            if (!isDir(dirName))
            {
                mkDir(dirName);
            }

            // Abrir archivo en modo append
            OFstream file(
                dirName/"centerOfMass.dat",
                IOstreamOption::ASCII,
                IOstreamOption::currentVersion,
                IOstreamOption::UNCOMPRESSED,
                IOstreamOption::APPEND
            );

            // Escribir tiempo y centro de masa
            file << time().value() << " " << com.x() << " " << com.y() << " " << com.z() << nl;
        #};
    }
```

## Función área de interfaz

```cpp
    interfaceArea
    {
        type            coded;
        libs            ("libutilityFunctionObjects.so");
        name            interfaceArea;

        codeExecute
        #{
            // Traemos el campo que vamos a utilizar alpha
            const volScalarField& alpha = mesh().lookupObject<volScalarField>("alpha.water");

            // fvc::snGrad(alpha) => Estamos calculando el gradiente de la normal de la superficie de la celda.
            // Luego mag está calculando solo la magnitud del vector y no su dirección.
            surfaceScalarField gradAlphaFace = mag(fvc::snGrad(alpha));

            // Magnitud del Área de las caras 
            const surfaceScalarField& faceArea = mesh().magSf();
            
            // Hacemos la integral del gradiente de alpha
            scalar interfaceArea = sum(gradAlphaFace * faceArea).value();

            // Crear carpeta si no existe
            fileName dirName = "postProcessing/interfaceArea";
            if (!isDir(dirName))
            {
                mkDir(dirName);
            }

            // Abrir archivo en modo append
            OFstream file2(
                dirName/"interfaceArea.dat",
                IOstreamOption::ASCII,
                IOstreamOption::currentVersion,
                IOstreamOption::UNCOMPRESSED,
                IOstreamOption::APPEND
            );

            // Escribir tiempo y centro de masa
            file2 << time().value() << " " << interfaceArea << nl;

        #};
    }
```

## Función fuerzas de la gota
```cpp
    dropForce
    {
        type            coded;
        libs            ("libutilityFunctionObjects.so");
        name            dropForce;

        codeExecute
        #{
            Info << "Calculando fuerza total sobre la gota..." << endl;

            // Constante sigma de la tensión superficial del agua.
            const dimensionedScalar sigma
            (
                "sigma",                            // nombre
                dimensionSet(1, 0, -2, 0, 0, 0, 0), // unidades: [kg m^-1 s^-2]
                0.072                               // valor numérico
            );

            // Campos necesarios serán alpha y la presión p-rho*g*h
            const volScalarField& alpha = mesh().lookupObject<volScalarField>("alpha.water");
            const volScalarField& p_rgh = mesh().lookupObject<volScalarField>("p_rgh");

            // Calculamos el campo vectorial en las caras de las celdas que 
            // apunta en dirección ortogonal a la superficie
            const surfaceVectorField& SfHat = mesh().Sf() / (mesh().magSf() + dimensionedScalar("tinyArea", dimArea, VSMALL));

            // Máscara para identificar la gota (umbral de alpha > 0.5)
            // Con esto diremos 1 donde tenemos agua y 0 donde no. (utilizando
            // el umbral de 0.5)
            const volScalarField mask = pos(alpha - 0.5);  // 1 donde hay gota

            // Interpolación a caras, llevamos los campos que son volumétricos
            // y calculados en los centros de las celdas, a las caras de las
            // celdas, esto se logra con la función fvc::interpolate(campo).
            surfaceScalarField mask_f   = fvc::interpolate(mask);
            surfaceScalarField alpha_f  = fvc::interpolate(alpha);
            surfaceScalarField p_rgh_f  = fvc::interpolate(p_rgh);

            // Presión sobre las caras de la gota, con esto calculamos las fuerzas
            // provocadas por presiones.
            surfaceVectorField forceP = -p_rgh_f * mesh().Sf(); // presión * normal * área

            // Tensión superficial: sigma * kappa * n * dA
            // calculamos las fuerzas por tensiones superficiales
            volScalarField magGradAlpha   = mag(fvc::grad(alpha));
            volVectorField nHat           = fvc::grad(alpha) / ( magGradAlpha + dimensionedScalar("tiny", magGradAlpha.dimensions(), VSMALL));
            volScalarField kappa          = -fvc::div(nHat);
            surfaceScalarField kappa_f    = fvc::interpolate(kappa);
            surfaceVectorField forceSigma = sigma * kappa_f * mesh().Sf(); // <-- Corrige aquí

            // Aplicar máscara (solo donde hay gota)
            forceP     *= mask_f;
            forceSigma *= mask_f;
            
            // Mostrar dimensiones (unidades)
            Info << "Dimensiones de forceP: "     << forceP.dimensions()     << endl;
            Info << "Dimensiones de forceSigma: " << forceSigma.dimensions() << endl;

            
            // Sumar e integrar numericamente los campos en toda la gota
            vector totalForce = sum(forceP + forceSigma).value();

            // Crear carpeta si no existe
            fileName dirName = "postProcessing/dropForce";
            if (!isDir(dirName))
            {
                mkDir(dirName);
            }

            // Guardar resultado
            OFstream file(
                dirName/"dropForce.dat",
                IOstreamOption::ASCII,
                IOstreamOption::currentVersion,
                IOstreamOption::UNCOMPRESSED,
                IOstreamOption::APPEND
            );

            file << time().value()
                 << " " << totalForce.x()
                 << " " << totalForce.y()
                 << " " << totalForce.z()
                 << endl;
            
        #};
    }
```

## Energía Cinética de la gota

```cpp
        kineticEnergyOfWater
    {
        type           coded;
        libs            ("libutilityFunctionObjects.so");
        name            kineticEnergyOfWater;

        codeExecute
        #{
            // Traemos los campos que vamos a utilizar
            const volVectorField& U        = mesh().lookupObject<volVectorField>("U");
            const volScalarField& rho      = mesh().lookupObject<volScalarField>("rho");
            const volScalarField& alpha    = mesh().lookupObject<volScalarField>("alpha.water");

            // Máscara para identificar la gota (umbral de alpha > 0.5)
            const volScalarField mask = pos(alpha - 0.5);  // 1 donde hay gota
            
            // Tomamos el alpha de la gota con el umbral aplicado.
            const volScalarField& alphaMask    = mesh().lookupObject<volScalarField>("alpha.water")*mask;		
            // Tomamos el rho de la gota con el umbral aplicado
            const volScalarField& rhoAlphaMask = rho * alphaMask;

            // Volumen de cada celda
            const auto& V = mesh().V();

            // Hacemos la integral del cuadrado de la velocidad
            // En donde magSqr es |U|^2 
            scalar KE = 0.5 * sum(rhoAlphaMask*V*magSqr(U)*mask);

            // Crear carpeta si no existe
            fileName dirName = "postProcessing/kineticEnergy";

            if (!isDir(dirName))
            {
                mkDir(dirName);
            }

            // Abrir archivo en modo append
            OFstream file(
                dirName/"kineticEnergyOfWater.dat",
                IOstreamOption::ASCII,
                IOstreamOption::currentVersion,
                IOstreamOption::UNCOMPRESSED,
                IOstreamOption::APPEND
            );

            // Escribir tiempo y energía cinética
            file << time().value() << " " << KE << nl;
        #}; 
    }
```

## Energía Cinética del aire

```cpp
kineticEnergyOfWind
    {
        type           coded;
        libs            ("libutilityFunctionObjects.so");
        name            kineticEnergyOfWind;

        codeExecute
        #{
            // Traemos el campo que vamos a utilizar
            const volVectorField& U        = mesh().lookupObject<volVectorField>("U");
            const volScalarField& rho      = mesh().lookupObject<volScalarField>("rho");
            const volScalarField& alpha    = mesh().lookupObject<volScalarField>("alpha.air");

            // Máscara para identificar la gota (umbral de alpha > 0.5)
            const volScalarField mask = pos(alpha - 0.5);  // 1 donde hay aire
            
            // Tomamos el alpha del viento
            const volScalarField& alphaMask    = mesh().lookupObject<volScalarField>("alpha.air")*mask;
            const volScalarField& rhoAlphaMask = rho * alphaMask;

            // Volumen de cada celda
            const auto& V = mesh().V();

            // Hacemos la integral del cuadrado de la velocidad
            scalar KE = 0.5 * sum(rhoAlphaMask*V*magSqr(U)*mask);//*magSqr(U)*V);

            // Crear carpeta si no existe
            fileName dirName = "postProcessing/kineticEnergy";

            if (!isDir(dirName))
            {
                mkDir(dirName);
            }

            // Abrir archivo en modo append
            OFstream file(
                dirName/"kineticEnergyOfWind.dat",
                IOstreamOption::ASCII,
                IOstreamOption::currentVersion,
                IOstreamOption::UNCOMPRESSED,
                IOstreamOption::APPEND
            );

            // Escribir tiempo y energía cinética
            file << time().value() << " " << KE << nl;
        #}; 
    }
```
# Conclusiones

Las funciones de post-procesamiento son increíblemente poderosas y si las combinamos con coded-functions, podemos calcular, crear o construir cualquier tipo de campo o métrica imaginable. Aunque muchas veces el hecho de ser tan flexibles las vuelve un poco ilegibles o difíciles de entender ya que nos adentramos en como OpenFOAM trata a los campos, a la malla y otras entidades para realizar cálculos. Es por eso que muchas veces puede resultar desafiante.

Animamos desde FoamFatalError a que piensen en distintas métricas primero y luego prueben realizar las coded functions para calcular dichas métricas.

¡Muchos Éxitos!