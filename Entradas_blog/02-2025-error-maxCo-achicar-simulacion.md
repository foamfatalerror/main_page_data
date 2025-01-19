# Introducción

El error que hoy vamos a estar abordando es el que suele suceder cuando achicamos una simulación de un tamaño macroscópico a un tamaño mas pequeño, como milimétrico o incluso peor, microscópico (en el orden de los $\mu m$).

Esto es muy común ya que en lineas generales partimos de casos tutoriales que tenemos dentro del mismo Open Foam y luego vamos ajustando mallado, campos, fluidos y dimensiones a nuestras condiciones mas particulares. Es posible que este error no lo tengas si partiendo de un caso tutorial, escalas a algo mas grande, sin embargo, yendo a algo muy pequeño seguramente surja este error o algo muy similar:

```bash
No Iterations 2
#0  Foam::error::printStack(Foam::Ostream&) at ??:?
#1  Foam::sigFpe::sigHandler(int) at ??:?
#2  ? in "/lib/x86_64-linux-gnu/libc.so.6"
#3  Foam::GAMGSolver::scale(Foam::Field<double>&, Foam::Field<double>&, Foam::lduMatrix const&, Foam::FieldField<Foam::Field, double> const&, Foam::UPtrList<Foam::lduInterfaceField const> const&, Foam::Field<double> const&, unsigned char) const at ??:?
#4  Foam::GAMGSolver::Vcycle(Foam::PtrList<Foam::lduMatrix::smoother> const&, Foam::Field<double>&, Foam::Field<double> const&, Foam::Field<double>&, Foam::Field<double>&, Foam::Field<double>&, Foam::Field<double>&, Foam::Field<double>&, Foam::PtrList<Foam::Field<double> >&, Foam::PtrList<Foam::Field<double> >&, unsigned char) const at ??:?
#5  Foam::GAMGSolver::solve(Foam::Field<double>&, Foam::Field<double> const&, unsigned char) const at ??:?
#6  Foam::fvMatrix<double>::solveSegregated(Foam::dictionary const&) at ??:?
#7  Foam::fvMatrix<double>::solve(Foam::dictionary const&) in "/opt/openfoam10/platforms/linux64GccDPInt32Opt/bin/compressibleInterFoam"
#8  Foam::fvMatrix<double>::solve() in "/opt/openfoam10/platforms/linux64GccDPInt32Opt/bin/compressibleInterFoam"
#9  ? in "/opt/openfoam10/platforms/linux64GccDPInt32Opt/bin/compressibleInterFoam"
#10  ? in "/lib/x86_64-linux-gnu/libc.so.6"
#11  __libc_start_main in "/lib/x86_64-linux-gnu/libc.so.6"
#12  ? in "/opt/openfoam10/platforms/linux64GccDPInt32Opt/bin/compressibleInterFoam"
Floating point exception (core dumped)
```

En este post vamos a estar abordando de donde nace este error, cual es el sentido físico del mismo y finalmente como solucionarlo (spoiler: Es bastante fácil, también es bastante fácil desorientarnos e ir por otros rumbos).

# Solución

Abordamos en primer lugar la solución para aquellos/as que sean mas ansiosos y quieran ir directo al meollo de la cuestión. La solución a este problema la vamos a encontrar en el archivo:

```
├── 0
│   ├── alpha.water.orig
│   ├── p.orig
│   ├── p_rgh.orig
│   ├── T.orig
│   └── U
├── Allclean
├── Allrun
├── constant
│   ├── g
│   ├── momentumTransport
│   ├── phaseProperties
│   ├── physicalProperties.air
│   └── physicalProperties.water
└── system
    ├── blockMeshDict
    ├── controlDict <-- (Acá está el problema)
    ├── decomposeParDict
    ├── fvSchemes
    ├── fvSolution
    └── setFieldsDict
```

En este archivo veremos que tendremos algo muy similar a lo siguiente:

```
/*--------------------------------*- C++ -*----------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     | Website:  https://openfoam.org
    \\  /    A nd           | Version:  10
     \\/     M anipulation  |
\*---------------------------------------------------------------------------*/
FoamFile
{
    format      ascii;
    class       dictionary;
    location    "system";
    object      controlDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

application     compressibleInterFoam;

startFrom       latestTime;

startTime       0;

stopAt          endTime;

endTime         0.5;

deltaT          0.00002;

writeControl    adjustableRunTime;

writeInterval   0.01;

purgeWrite      0;

writeFormat     binary;

writePrecision  8;

writeCompression off;

timeFormat      general;

timePrecision   10;

runTimeModifiable yes;

adjustTimeStep  yes;

maxCo           0.5; <-- (Aca está el problema)
maxAlphaCo      0.5;
maxDeltaT       1;


// ************************************************************************* //
```

Reemplazando este `0.5` por un `0.05` o valores mas pequeños al original (podemos ir probando), comenzaremos a ver que por un lado la simulación tarda un poco mas pero también veremos que comienza a no fallar.

# Explicación de la falla

## ¿Que #$%#% es el número de Courant?

El número de **Courant** es una condición o **criterio de convergencia** que ponemos a fenómenos físicos que se mueven a través del espacio para lograr "capturar" su movimiento. 

Digamos que en palabras mas abreviadas, nos dice que tan pequeño debería ser nuestro paso temporal para poder sacar una "foto" del evento físico y que en la siguiente "foto" que tomemos ese sistema/evento/objeto físico ya se nos haya salido de cuadro donde estamos enfocando. Conceptualmente es esto aunque quizá sea mas entendido con el ejemplo de la próxima sección.
 
La ecuación que lo define es la siguiente:

$$
C=\frac{\mu}{\Delta x/\Delta t}
$$


En donde $C$ es el número de Courant, $\Delta x$ es espaciado del mallado, $\mu$ es la velocidad del objeto/evento/sistema físico y $\Delta t$ es el espaciado temporal de la simulación o cada *cuanto tomamos la foto*.
## Ejemplo de aplicación del Número de Courant

Para poder ejemplificar esta situación del evento/objeto/sistema físico que queremos **Fotografiar** vamos a proponer una pequeña animación.


[Animación de ejemplo](https://www.youtube.com/embed/dQ5JoI2wQy0?si=Og5N7cChSoNXDjyc)]

En este ejemplo vemos claramente que para el primer lanzamiento de la bola, podemos captar su movimiento entre foto y foto, debido que a **cae** dentro de nuestra pantalla o campo de visión (Que sería como la celda en una simulación), esto nos permite poder mesurar correctamente que sucede.

Sin embargo para el segundo lo que sucede es que la velocidad de la bola es tan grande que "escapa" de nuestra celda o campo de visión. Esto provoca que no podamos ver o medirlo. Esto provoca un error serio dentro de Open Foam y es justamente el que nos lanza cuando hacemos alguna de las siguientes cosas:

- Achicar un dominio (Sería equivalente a achicar $\Delta x$)
- Agrandar el tiempo que dejamos pasar entre paso temporal y paso temporal (Agrandar $\Delta t$)


