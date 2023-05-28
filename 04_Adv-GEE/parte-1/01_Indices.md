---
layout: page
title: Indices
parent: Google Earth Engine Avanzado - Parte 1
nav_order: 1
---

## Script
El script completo que se usará en esta sección esta disponible [aquí]().

# Índices y operaciones matématicas

En sesiones anteriores vimos como extraer información de productos disponibles en GEE. Uno de ellos fue el producto de MODIS Terra ["MODIS/061/MOD13Q1"](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD13Q1) que entrega indices de vegetación (NDVI y EVI) cada 16 días a 250m / pixel.
Sin embargo, podemos usar cualquier colección con datos multiespectrales para calculor nuestros propios índices, ya sea para obtener mejor resolución espacial o temporal, o información que no esté disponible en GEE.

Para los siguientes ejemplos usaremos la colección de [Landsat-8 L2](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2#description), para lo cual vamos a precargar algunas funciones primero:

```javascript
// Funciones precargadas:

// Función enmascarar nubes Landsat-8:
function maskL8clouds(image) {
  var qa = image.select('QA_PIXEL'); //Select the QA band

  // Bits 5 is clouds.
  var cloudBitMask = 1 << 3;  

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0);

  return image.updateMask(mask);
}

// Función para aplicar factores de escala Landsat-8:
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true);
}
```


Adicionalmente, vamos a preparar una colección de imágenes, filtrando, aplicando factores de escala y limpiando nubes:

```javascript
// Polígono de Colombia
var colombia = ee.FeatureCollection("USDOS/LSIB/2017").filter(ee.Filter.eq('COUNTRY_NA','Colombia'));

// Cargar imagenes Landsat-8 y pre-procesar:
var coleccion = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
                .filterBounds(colombia)
                .filterDate('2022-01-01','2022-12-31')
                .filter(ee.Filter.lt('CLOUD_COVER',30))
                .map(maskL8clouds)
                .map(applyScaleFactors);

// Generar mosaico compuesto y recortar:
var compuesto = coleccion.median().clip(colombia);

// Visualizar mapa:
Map.addLayer(compuesto,{bands:['SR_B4','SR_B3','SR_B2'],min:0,max:0.2},'Compuesto');
```

Vamos a obtener un mosaico como el siguiente, con algunos vacios de información en la zona Andina de Colombia, pero con buena calidad en sur, este y norte.

<img align="center" src="../../images/adv-gee/01_fig1.png" vspace="10" width="500">

## NDVI - Normalized Difference Vegetation Index:

El índice de diferencia de vegetación normalizada (NDVI) tiene un rango de valores entre -1 a +1. Típicamente, cuando hay valores negativos existe alta probabilidad que se trate de un cuerpo de agua. Por otro lado, si los valores son cercanos a +1 es probable que se trate de vegetación muy densa. Cuando el NDVI es cercano a cero es probable que se trate de un área urbana.

Para calcular el NDVI usamos la siguiente fórmula:

$$\frac{(NIR-Rojo)}{(NIR+Rojo)}$$

La relación entre las bandas NIR (infrarrojo cercano o Near-Infrared) y Rojo van a proporcionar un índice de vegetación.

<img align="center" src="../../images/adv-gee/01_fig2.jpg" vspace="10" width="500">

```javascript
// Calcular NDVI:
// NDVI = (NIR - RED) / (NIR + RED)
var NIR = compuesto.select('SR_B5');
var Red = compuesto.select('SR_B4');
var ndvi = NIR.subtract(Red).divide(NIR.add(Red));

// Paleta de verdes:
var palette = [
    'FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163', '99B718', '74A901',
    '66A000', '529400', '3E8601', '207401', '056201', '004C00', '023B01',
    '012E01', '011D01', '011301'];

// Visualizar:
Map.addLayer(ndvi, {'palette': palette}, "NDVI");
```

Otra alternativa para calcular indices normalizados es usar la función `normalizedDifference`. Para este ejemplo, nuestro NDVI podrías ser calculado como `var ndvi = composite.normalizedDifference(['SR_B4', 'SR_B3'])`.

<img align="center" src="../../images/adv-gee/01_fig1.png" vspace="10" width="500">