# Cuantificación de volumen captable en diques o cosechadora de agua de temporal usando R
Miguel Armando López Beltrán

miguel.armandolb@uas.edu.mx

Universidad Autónoma de Sinaloa

22 de Mayo de 2024

## Objetivo
Proporcionar a los estudiantes las habilidades necesarias para generar un script en el lenguaje de programación en R y aplicarlo específicamente en el análisis y cuantificación del volumen de agua que puede ser almacenado en una cosechadora de agua.


## Configuración de bibliotecas y descarga de datos


### Descripción de los paquetes
httr
: Permite descargar archivos desde R a traves de solicitudes a servidores web.

readxl
: Permite importar las hojas de calculo de excel.

rgdal
: Permite usar las funciones para trabajar con datos vectoriales.

sf
: Permite la manipulación de los datos vectoriales.

raster
: Permite usar las funciones de cargar, manipular, visualizar y analizar los datos para el procesamientos y análisis de datos raster. 

rasterVis
: Paquete complementario de raster que sirve para la visualización de éstos.

rgl
: Se utiliza para la creación de gráficos en 3D.

gstat
: Ofrece funciones de análisis espacial.

magrittr
: Ofrece la utilización de optimización de codigo para mejorar la estructura del código (%>%).

ggplot2
: Crea gráficos de alta calidad.

El código realiza una verificación del paquete específico, si se encuentra instalado solo lo cargara, en caso contrario realizara el procedimiento de instalarlo y cargarlo en la sesión activa.

```{r}
if(!require(httr)){
  install.packages("httr")
  require(httr)}else{library(httr)}

if(!require(readxl)){
  install.packages("readxl")
  require(readxl)}else{library(readxl)}

if(!require(rgdal)){
  install.packages("rgdal")
  require(rgdal)}else{library(rgdal)}

if(!require(sf)){
  install.packages("sf")
  require(sf)}else{library(sf)}

if(!require(raster)){
  install.packages("raster")
  require(raster)}else{library(raster)}

if(!require(rasterVis)){
  install.packages("rasterVis")
  require(rasterVis)}else{library(rasterVis)}

if(!require(rgl)){
  install.packages("rgl")
  require(rgl)}else{library(rgl)}

if(!require(gstat)){
  install.packages("gstat")
  require(gstat)}else{library(gstat)}

if(!require(magrittr)){
  install.packages("magrittr")
  require(magrittr)}else{library(magrittr)}

if(!require(ggplot2)){
  install.packages("ggplot2")
  require(ggplot2)}else{library(ggplot2)}

```


### Importación de datos
El código descarga el archivo en forma temporal, su contenido es un identificador y coordenadas X, Y, Z. 

En este caso práctico se descargara directamente desde github:
```{r}
#Directorio donde se localiza el archivo en formato de excel
url<-"https://github.com/malbmx/Taller7cneggg/raw/main/Puntos.xlsx"

#Creamos un archivo temporal
temporal<-tempfile(fileext = ".xlsx")
GET(url, write_disk(temporal, overwrite = TRUE))

#Importamos a R y generamos un data frame que contendrá los datos
Datos<-as.data.frame(read_excel(temporal))

head(Datos)
```


## Análisis exploratorio de datos 

### Limpieza y preprocesamiento de los datos
Anteriormente, el archivo fue importado a la sesión en R con la función *read_excel*.

### Identificación del contenido de datos
En ocasiones es necesario ver si existen datos faltantes *NA* y eliminar las lineas inecesarias. Tambien se verifica si hay filas duplicadas y las elimina si es necesario. 

#### Búsqueda de datos faltantes
```{r}
if(sum(is.na(Datos))>0){
    cat("\nPresenta datos faltantes o errores en el tipo de dato...\n")
    cat("\nFilas eliminadas: ", sum(is.na(Datos))," de ", nrow(Datos),
        "registros (",round((sum(is.na(Datos))*100/nrow(Datos)),4),"%)\n")
    Datos<-drop_na(Datos)
    }else{cat("Sin datos faltantes...\n")}
```
#### Búsqueda de datos duplicados
```{r}
verif<-table(duplicated(Datos))
  is.na(verif[2])
  if (is.na(verif[2])==FALSE) {
    Datos[!duplicated(Datos),]
    print("Presenta datos duplicados...\n")
    Datos<-Datos%>%distinct()}else{
    cat("\nNo presenta datos duplicados...\n")
    }
```

### Identificación de valores atípicos

Aplicamos un diagrama de cajas (boxplot) para analizar y resumir la variabilidad de un conjunto de datos, lo que permite identificar valores atipicos.

```{r}
boxplot(Datos$Z)
```


### Visualización gráfica
Una manera de visualizar la información geoespacial en la ubicación de los puntos, correlacionamos las coordenadas X y Y.

```{r}
plot(Datos$Y~Datos$X)
```


### Estadística descriptiva
El análisis descriptivo es una herramienta que proporciona una compresión básica y esencial de los datos.
```{r}
summary(Datos)
hist(Datos$Z)
mean(Datos$Z)
sd(Datos$Z, na.rm=TRUE)

```

En este ejemplo solo nos interesa conocer la media y su desviación estándar de la altura (Z) para tener una noción de como se puede distribuir. Además agregamos el histograma para la frecuencia de los datos.




## Preparación de un raster para los datos

### Dimensiones de la imagen

Obtenemos el valor mínimo y máximo en las coordenadas X y Y. EL objetivo es obtener los vertices de las coordendadas del raster.   

```{r}
x.range<-as.numeric(range(Datos$X))
x.range
```

```{r}
y.range<-as.numeric(range(Datos$Y))
y.range
```


#### Dataframe bidimensional
Con los rangos determinados en las coordenadas X y Y, se procede a crear una matriz donde se guardaran los datos interpolados. El comando **seq** crea una secuencia desde un valor inicial a un valor final, asimismo, asignamos la resolución espacial que tendrá cada pixel en metros dentro de la matriz.
Con la función *head* visualizaremos solo los primeros 5 datos. 
```{r}
# Definir la resolución espacial.
res_pixel<-0.1

# Generamos la secuencia numérica de coordenadas para la matriz. 
x_seq<-seq(x.range[1], x.range[2], by=res_pixel)
head(x_seq)
y_seq<-seq(y.range[1], y.range[2], by=res_pixel)
head(y_seq)
```


Con los valores de la dimensión se crea un nuevo dataframe bidimensional. Con la función *expand.grid* toma en cuenta los vectores generados anteriormente (*x_seq* y *y_seq¨*):
```{r}
# Dataframe bidimensional destinado para almacenar los resultados de la interpolación
grd<-expand.grid(x_seq, y_seq)
plot(grd)
```

### Transformación a datos geoespaciales
Se transforma los datos a datos geoespaciales con un sistema de referencia usando EPSG: 32613 (UTM-13 N). 

```{r}
Datos
# Se definen las coordenadas
coordinates(Datos)<-~X+Y
# Se verifica que ahora se encuentra en un SpatialPointDataFrame
Datos
#Asignamos la proyección
crs(Datos)<-CRS("+init=epsg:32613")
# Ahora contiene un CRS
Datos

# Se realiza lo mismo que a los datos
coordinates(grd)<-~Var1+Var2
crs(grd)<-CRS("+init=epsg:32613")
grd

# Configuramos las propiedades cuadricula
gridded(grd)<-TRUE
fullgrid(grd)<-TRUE

# Visualizamos la matriz
plot(grd, cex=2, col="gray", main="Matriz generada para almacenar datos interpolados.")

```

Visualizamos donde se encuentran los puntos:
```{r}
plot(grd, cex=2, col="gray", main="Matriz generada para almacenar datos interpolados.")
points(Datos, pch=1, col="red", cex=1)
```

### Interpolación de distancia inversa ponderada
```{r}

# Configuramos el valor de la ponderación al modelo
W<-2

# Usamos la configuración para generar la interpolación de distancia inversa ponderada.
idw_model<-gstat(formula=Z~1, data = Datos, nmax=length(Datos$Z), set=list(idp=W))

# Generamos el modelo.
modelo<-predict(object=idw_model, newdata=grd)

# Rasterizamos la imagen
modelo<-raster(modelo)

# Visualización
plot(modelo, main="Modelo digital de elevación de un dique")
```

Agregamos colores.

```{r}
plot(modelo, main="Modelo digital de elevación de un dique.", col=topo.colors(10))
```



### Validación de la interpolación

Para la validación se utilizo la raíz del error medio cuadrático:
\begin{equation}
RMSE= \sqrt{\dfrac{ \sum (Zi-Zp)^2}{N}}
\end{equation}

Lo siguiente implementamos el algoritmo para el calculo de RMSE, reflejando la magnitud del error promedio entre los valores predichos y el valor real observado. Un valor bajo indica un mejor rendimiento del modelado.

```{r}
# Extraemos los valores del pixel
valores_predichos<-extract(modelo, Datos$Z)
valor_real<-Datos$Z
# Aplicamos la ecuación del RMSE
rmse<-sqrt(mean((valores_predichos-valor_real)^2))
#rmse
cat("Ponderación: ",paste0(W),
    "\nRMSE= ", paste0(rmse))
```



### Creación de curvas de nivel

Se crean curvas de nivel con la intención de generar un nivel máximo de agua. 

```{r}
intervalo<-seq(min(Datos$Z), max(Datos$Z), by=0.25)
curvas<-contour(modelo, levels=intervalo)
```

 
En este caso seleccionamos y redondeamos el valor de 90.39143 y seleccionamos la curva de 90.4 m.

```{r}
levelplot(modelo, contour=TRUE, col.regions=topo.colors(20))
```


```{r}
plot3D(modelo, col=topo.colors(10))
```


### Cuantificación del volumen total de agua a capacidad máxima de la cosechadora de agua
Calculamos la lámina de agua en cada pixel, para ello aplicaremos la siguiente ecuación:

$$La = Curva-Pixel $$
Donde el valor de la curva es el valor seleccionado anteriormente (90.4), y el pixel, los valores del modelo.

```{r}
# Obtenemos los valores del modelo
valores_raster<-getValues(modelo)

# Definimos el nivel máximo de agua.
Curvamaxima<-90.4

#Aplicamos la ecuación para la lámina de agua
Lamina<-Curvamaxima-valores_raster
# Creamos un raster basado en el modelo 
raster_Lamina<-modelo
#Asignamos los valores a los pixeles del resultado de la lamina
values(raster_Lamina)<-Lamina
plot(modelo)
plot(raster_Lamina)
# Ajustes de valores negativos, establece que todos los valores menores a 0 se ajustan a 0.
Lamina[Lamina<0]<- 0

values(raster_Lamina)<-Lamina
plot(raster_Lamina,  col=rev(topo.colors(10)), main="Lámina de agua")
```

### Estimación del volumen

Para la estimación del volumen se aplica un simple ecuación tomando en cuenta el área y la altura (lamina) de agua.

$$Vol= R_x * R_y * La$$
$Vol$: Volumen en el pixel.
$R_x$ Resolución espacial en x.
$R_y$ Resolución espacial en y.
$La$ Lámina de agua.  

```{r}
# Obtenemos la resolución
resolucion<-res(raster_Lamina)
r_x<-resolucion[1]
r_y<-resolucion[2]
# Aplicamos la ecuación
Volumen<-getValues(raster_Lamina)*r_x*r_y
# Sumamos todos los voluemens en los pixeles.
VolumenTotal<- sum(Volumen, na.rm=TRUE)
cat("Volumen total captable: ", VolumenTotal, "m^3")
```



### Generación cartográfica

```{r}
cartografia<-levelplot(modelo,
                       col.regions=topo.colors(20),
                       main="Modelo digital de elevación")
cartografia
plot(modelo, main="Modelo digital de elevación (msnm)",
     xlab="Longitud",
     ylab="Latitud",
     sub="Proyección UTM-13N"
     )

```

### Exportar archivos
Ya finalizado exportamos los archivos de datos geoespaciales.

```{r}
# Exportar el dato raster
writeRaster(modelo, "~/Taller/mde.tif",format="GTiff", overwrite=TRUE )

# Exportar el dato vectorial
writeOGR(Datos, "~/Taller/","Puntos vectoriales", driver="ESRI Shapefile", overwrite_layer = TRUE)
```

Nota: Debe de existir el directorio. 






## Práctica 2
**Modificamos el valor *W*** y seleccionaremos el mejor modelo para obtener el volumen. 

## Finalización del taller

- La cantidad de volumen va depender de la altura de agua que consideremos como la curva máxima de agua. En consecuencia aumenta o disminuye el volumen total.

- También dependerá del modelo a utilizar para realizar la interpolación correspondiente.

- Para mejorar la cuantificación se recomienda que durante el levantamiento topográfico se designe el valor de altura máxima para omitir la selección de la curva de nivel. 

- Para evitar el ruido externo del área del dique (considerando que también se realizara levantamiento fuera del área) es recomendable utilizar una máscara del área del dique.



