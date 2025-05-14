#fluid-properties 
## Introducción

Para aplicaciones específicas, uno podría necesitar simular no solo determinado fluido ,distinto al $H_2O$ o al $O_2$, sino ademas definir sus propiedades termofísicas variables en función de la temperatura.

Si miramos uno de los casos [tutoriales](https://github.com/OpenFOAM/OpenFOAM-10/blob/master/tutorials/heatTransfer/chtMultiRegionFoam/heatedDuct/constant/fluid/physicalProperties) de OpenFOAM, veremos como en el archivo *physicalProperties* en muchos casos tiene su fluido con propiedades termofísicas constantes:

```cpp
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
    location    "constant/fluid";
    object      physicalProperties;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

thermoType
{
    type            heRhoThermo;
    mixture         pureMixture;
    transport       const;
    thermo          hConst;
    equationOfState rhoConst;
    specie          specie;
    energy          sensibleEnthalpy;
}

mixture
{
    // Water
    specie
    {
        molWeight       18;
    }
    equationOfState
    {
        rho             1000;
    }
    thermodynamics
    {
        Cp              4181;
        Hf              0;
    }
    transport
    {
        mu              959e-6;
        Pr              6.62;
    }
}
// ************************************************************************* //
```

En este tutorial veremos brevemente como hacer una simulación con la densidad variable en función de la temperatura.
## Implementación

### Datos

Para conocer como varía la densidad a una determinada presión de interes, y en un determinado rango de temperaturas que sean los de nuestro caso, pueden acudir a [distintas fuentes](https://www.engineeringtoolbox.com/water-density-specific-weight-d_595.html). En nuestro caso, voy a mostrar el ejemplo de una base de datos que suelo usar habitualmente porque tiene valores precisos de propiedades para distintos fluidos.  

Luego de seleccionar la tabla de su interes, pueden (luego de convertir las unidades a MKS), generar una tabla sencilla en txt o csv de la siguiente forma:

```txt
0.1	0.9998495	999.85	1.9400	62.4186	8.3441	9.8052	62.419	-0.68
1	0.9999017	999.90	1.9401	62.4218	8.3446	9.8057	62.422	-0.50
4	0.9999749	999.97	1.9403	62.4264	8.3452	9.8064	62.426	0.003
10	0.9997000	999.70	1.9397	62.4094	8.3429	9.8037	62.409	0.88
15	0.9991026	999.10	1.9386	62.3719	8.3379	9.7978	62.372	1.51
20	0.9982067	998.21	1.9368	62.3160	8.3304	9.7891	62.316	2.07
25	0.9970470	997.05	1.9346	62.2436	8.3208	9.7777	62.244	2.57
30	0.9956488	995.65	1.9319	62.1563	8.3091	9.7640	62.156	3.03
35	0.9940326	994.03	1.9287	62.0554	8.2956	9.7481	62.055	3.45
40	0.9922152	992.22	1.9252	61.9420	8.2804	9.7303	61.942	3.84
45	0.99021	        990.21	1.9213	61.8168	8.2637	9.7106	61.817	4.20
50	0.98804	        988.04	1.9171	61.6813	8.2456	9.6894	61.681	4.54
55	0.98569	        985.69	1.9126	61.5346	8.2260	9.6663	61.535	4.86
60	0.98320	        983.20	1.9077	61.3792	8.2052	9.6419	61.379	5.16
65	0.98055	        980.55	1.9026	61.2137	8.1831	9.6159	61.214	5.44
70	0.97776	        977.76	1.8972	61.0396	8.1598	9.5886	61.040	5.71
75	0.97484	        974.84	1.8915	60.8573	8.1354	9.5599	60.857	5.97
80	0.97179	        971.79	1.8856	60.6669	8.1100	9.5300	60.667	6.21
85	0.96861	        968.61	1.8794	60.4683	8.0834	9.4988	60.468	6.44
90	0.96531	        965.31	1.8730	60.2623	8.0559	9.4665	60.262	6.66
95	0.96189	        961.89	1.8664	60.0488	8.0274	9.4329	60.049	6.87
100	0.95835	        958.35	1.8595	59.8278	7.9978	9.3982	59.828	7.03
```
En el siguiente paso, usaremos solo la primer y la tercer columna, que contienen los valores de temperatura y densidad.
### Carga de datos

Con este script de python, podríamos generar dos array con los valores de la temperatura y su densidad correspondiente:

```python
import matplotlib.pyplot as plt
import numpy as np

# Crear listas vacías para almacenar los datos de temperatura y densidad
temperatura = []
densidad = []

# Abrir el archivo "density.txt" en modo lectura
with open("density.txt", "r") as archivo:

	# Leer cada línea del archivo
	for linea in archivo:
	
		# Dividir la línea en columnas utilizando espacios en blanco como separador
		columnas = linea.strip().split()
			
		# Convertir el primer elemento (columna 0) a float y sumar 273.15
		temperatura.append(float(columnas[0]) + 273.15)
		
		# Convertir el tercer elemento (columna 2) a float
		densidad.append(float(columnas[2]))

# Convertir las listas a arrays de numpy
temperatura = np.array(temperatura)
densidad = np.array(densidad)

# Imprimir los vectores resultantes
print("Temperatura:", temperatura)
print("Densidad:", densidad)
```
---
```
Temperatura: [273.25 274.15 277.15 283.15 288.15 293.15 298.15 303.15 308.15 313.15
 318.15 323.15 328.15 333.15 338.15 343.15 348.15 353.15 358.15 363.15
 368.15 373.15]
Densidad: [999.85 999.9  999.97 999.7  999.1  998.21 997.05 995.65 994.03 992.22
 990.21 988.04 985.69 983.2  980.55 977.76 974.84 971.79 968.61 965.31
 961.89 958.35]
 ```

Con los array ya cargados, se puede proceder buscar la curva que mejor ajuste estos valores, printearla, y como verificación extra, graficarla tambien:

```python
from scipy.optimize import curve_fit
from scipy.stats import linregress
import math

# Definir una función logarítmica
def func(x, a, b, c, d):
	return a + b*x + c*x**2

# Ajustar la función logarítmica a los datos
params, covariance = curve_fit(func, temperatura, densidad)

# Parámetros ajustados
a, b ,c, d = params

# Crear el gráfico de puntos
plt.scatter(temperatura, densidad , label='Datos')
plt.xlabel('Temperatura')
plt.ylabel('Densidad')
plt.title('Gráfico de Puntos y Ajuste')

# Crear la función logarítmica ajustada
x_fit = np.linspace(273, 380, 1000) # Define el rango deseado en el eje x
y_fit = func(x_fit, a, b,c,d)

# Graficar la función logarítmica ajustada
plt.plot(x_fit, y_fit, 'r-', label=f'Ajuste: a={a:.4f}, b={b:.4f}, c={c:.8f}')
plt.legend()
plt.grid(True)

# Mostrar el gráfico
plt.show()

print(a,b,c)
```
---
```
749.090158602236 1.906677176442396 -0.0036111040781610514
```
---
![](https://raw.githubusercontent.com/foamfatalerror/main_page_data/refs/heads/main/imagenes_blog/202505_propiedades_termofisicas_variables/fiteo.png)

El ajuste se traduce en la siguiente ecuación:
$$
\rho(T) = 749.0901586 +1.906677T -3.6111\times10^{-3}T^2
$$


### Implementación en OpenFOAM

> **_ACLARACIÓN:_** En este ejemplo, se aplico a la versión 10 de [openfoam.org](https://openfoam.org/).

La forma de cargar ahora la Eq. $(1)$ en OpenFoam, es usando la siguiente lógica:

```cpp
rhoCoeffs<8>    ( c b a 0 0 0 0 0 );
```


Cabe destacar que la precisión en la simulación estará luego ligada tambien a la precisión usada en los parametros "a,b,c" usados en physicalProperties.

```cpp
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
    location    "constant/fluid";
    object      physicalProperties;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

thermoType
{
    type            heRhoThermo;
    mixture         pureMixture;
    transport       const;
    thermo          hConst;
    equationOfState icoPolynomial;
    specie          specie;
    energy          sensibleEnthalpy;
}

mixture
{
    // Water
    specie
    {
        molWeight       18;
    }
    equationOfState
    {
        rhoCoeffs<8>    ( 749.0901442669926 1.90667726641442 -0.003611104217991187 0 0 0 0 0 );
    }
    thermodynamics
    {
        Cp              4181;
        Hf              0;
    }
    transport
    {
        mu              959e-6;
        Pr              6.62;
    }
}

// ************************************************************************* //
```
## Resultados
L siguiente imagen muestra la visualización del campo de densidades en un flujo de agua dentro de un ducto. El flujo va en dirección de derecha a izquierda, y la imagen muestra una sección transversal a la mitad del ducto y a lo largo del canal. Inicialmente, el flujo entra con $\rho=1000 kg/m^3$, pero al calentarse por efecto del bloque gris inferior que se observa, la densidad al observarla en el estado estacionario se modificó y ahora, cerca del calentador , podemos observar una densidad mínima de $\rho=990 kg/m^3$.
![](https://lh7-rt.googleusercontent.com/slidesz/AGV_vUeO_a6wcoT4vNxLKjzgSlGgePM7vR87TR0bOD0h3WRXbLVQ-YDTqZ-hMn0GxCyUABzkFzcaMQxeZUvJdsFbRtyFODgvAy3Pf5lF-MRzDIFjwh4qAsJ578RutkhJVPofji8nAkfzQiATXa6nTq2q8HHe25rforxlQRQ5UWL5CxBSvQ=s2048?key=AXNzHZTJBbCa1kVL4DQI2A)

## Ejemplo de aplicación

En el siguiente [paper](https://arxiv.org/pdf/2402.09931) se encuentra un ejemplo de modificación de las propiedades termofísicas no solo de la densidad, sino tambien de la conductividad térmica, de la capacidad calorífica y tambien de la viscosidad del fluido.


