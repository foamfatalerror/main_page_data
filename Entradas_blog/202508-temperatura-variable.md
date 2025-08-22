# Introducción

En este tutorial vamos a estar explorando específicamente como variar el valor de un campo en un patch como condición de contorno.

Este tipo de variación en las condiciones de contorno nos sirve mucho para simular comportamientos físicos no lineales o no constantes en los bordes de nuestro sistema. Un ejemplo de esto podría ser el calor generado y retirado por el ciclo noche-dia en un sistema cerrado. Tipicamente pareciendosé mucho a una función seno de temperatura en las paredes de un sistema con gran inercia térmica en las mismas. También tenemos sistemas en donde la presión o la velocidad de un flujo son oscilantes, o tienen comportamientos erráticos que pueden ser representados por alguna función matemática.

## Consideraciones
- Utilizaremos el solver rhoPimpleFoam.
- Utilizaremos la versión 10 del OpenFOAM.org
- Variaremos el campo temperatura para esta ocasión.
- El sistema será 2D para una simplificación la explicación
- El sistema será transitorio y no estacionario (tambien el tipo de simulación)
# Temperatura Variable en un Patch

## Lo que haremos

Para poder escribir una condición de contorno variable, vamos a tomar el siguiente caso con sus archivos:

```bash
├── 0
│   ├── p
│   ├── T <-- Alteraremos esta condición de contorno
│   ├── tracer
│   ├── T.solid
│   └── U
├── Allclean
├── Allrun
├── constant
│   ├── fvModels
│   ├── momentumTransport
│   ├── physicalProperties
│   ├── physicalProperties.solid
│   └── thermophysicalTransport
└── system
    ├── blockMeshDict
    ├── controlDict
    ├── fvSchemes
    ├── fvSolution
    └── generateAlphas
```

Lo que haremos es tomar este caso que va de un tiempo 0 a 0.03 segundos y le pondremos una temperatura en el borde inlet dependiente del tiempo. En particular la función de temperatura en función del tiempo, será:

$$
T=T_{mean} + A_{mp} \cdot sen(\omega \cdot t)
$$
En donde:
- $T_{mean} = 300$
- $A_{mp}=100$
- $\omega=2\cdot \pi / 0.03$
y el tiempo será variable en cada paso temporal de la siguiente manera

![Imagen](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202508-temperatura-variable/newplot.png)

## Modificación de la condición de contorno

La palabra clave para introducir un tipo de condición de contorno en donde podamos alterar los valores como si fuera código es **codedFixedValue** y se estructura como se ve en el siguiente ejemplo:

```cpp
    inlet // Patch sobre el cual vamos a aplicar la condición
    {
        type            codedFixedValue; // Condición de contorno
        value           uniform 300; // Valor inicial del campo
        name            tsin; // Nombre de la función

        code
        #{
            // Definición de parámetros para la función seno
            const scalar tmean = 300;
            const scalar amp   = 100;
            const scalar t     = this->db().time().value();
            const scalar tau   = 0.03;
            const scalar omega = 2.0 * constant::mathematical::pi / tau;
            const scalar value = tmean + amp * sin(omega * t);

            // Asignar el valor para un tiempo t, a el patch.
            this->operator==(value);
        #};
    }
```

Una vez hecho esto ya tendremos la condición de contorno moviendose como nosotros deseamos. 

También al estar escrito en C++ todo lo que esta dentro de **code** , podremos introducir Ifs, whiles, for, else, switch, etc. Es decir cualquier tipo de estructura de control deseada. Por ejemplo con un if, podríamos activar el comportamiento senoidal posterior a cierto tiempo deseado. 

También podríamos relacionar campos entre si, creando dependencias entre campos como la presión y la temperatura u otros parámetros que deseemos.

## Aclaraciones
1. El valor que pongamos en `operator` será el que fijará la condición de contorno. Es importante entender que el valor será constante a lo largo de toda la geometría del patch.

## Resultados de la simulación
En el tiempo inicial la función seno está t=0, con lo cual no vemos la onda de temperatura, luego para el tiempo final, podemos ver como la onda de temperatura avanza por el dominio desde el inlet, que está el lado izquierdo.

**Tiempo inicial**
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202508-temperatura-variable/tiempo-0.png)


**Tiempo Final**
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202508-temperatura-variable/tiempo-final.png)


# Material Adicional
1. [Link al caso](https://drive.google.com/drive/folders/1U6ELyMpl3tnDJzNIsWVh35iD2xsmbdTw?usp=sharing)