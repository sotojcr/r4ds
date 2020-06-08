# Muchos modelos

## Introducción

En este capítulo vas a aprender tres ideas poderosas que te van a ayudar a trabajar fácilmente con un gran número de modelos:

1. Usar muchos modelos simples para entender mejor conjuntos de datos complejos.

1. Usar columnas-lista (_list-columns_) para almacenar estructuras de datos arbitrarias en un _data frame_.
 Esto, por ejemplo, te permitirá tener una columna que contenga modelos lineales.

1. Usar el paquete __broom__, de David Robinson, para transformar modelos en datos ordenados.
 Esta es una técnica poderosa para trabajar con un gran número de modelos
 porque una vez que tienes datos ordenados, puedes aplicar todas las técnicas que
 has aprendido anteriormente en el libro.

Empezaremos entrando de lleno en un ejemplo motivador usando datos sobre la esperanza de vida alrededor del mundo. Es un conjunto de datos pequeño pero que ilustra cuán importante puede ser modelar para mejorar tus visualizaciones. Utilizaremos un número grande de modelos simples para separar algunas de las señales más fuertes y así poder ver las señales sutiles que permanecen. También veremos cómo las medidas de resumen de los modelos nos pueden ayudar a encontrar datos atípicos y tendencias inusuales.

Las siguientes secciones ahondarán en más detalles acerca de cada una de estas técnicas:

1. En [columnas-lista] aprenderás más acerca de la estructura de datos _columna-lista_
 y por qué es válido poner listas en _data frames_.

1. En [creando columnas-lista] aprenderás las tres maneras principales en las que
 crearás columnas-lista.

1. En [simplificando columnas-lista] aprenderás cómo convertir columnas-lista de vuelta
 a vectores atómicos regulares (o conjuntos de vectores atómicos) para que puedas trabajar
 con ellos más fácilmente.

1. En [haciendo datos ordenados con broom] aprenderás sobre el conjunto de herramientas completo
 provisto por __broom__ (_escoba_, en inglés) y verás cómo puede ser aplicado a otros tipos de
 estructuras de datos.

Este capítulo es en cierta medida aspiracional: si este libro es tu primera introducción a R, este capítulo probablemente será un desafío. Requiere que tengas profundamente internalizadas ideas acerca de modelado, estructuras de datos e iteración. Así que no te preocupes si no lo entiendes --- solo aparta este capítulo por un par de meses, y vuelve cuando quieras ejercitar tu cerebro.

### Prerrequisitos

Trabajar con muchos modelos requiere muchos de los paquetes del __tidyverse__ (para exploración de datos, manejo y programación) y __modelr__ para facilitar el modelado.


```r
library(modelr)
library(tidyverse)
```

## gapminder

Como motivación para entender el poder de muchos modelos simples, vamos a mirar los datos de "gapminder". Estos datos fueron popularizados por Hans Rosling, un doctor y estadístico sueco. Si nunca has escuchado de él, para de leer este capítulo ahora mismo y revisa algunos de sus videos! Es un fantástico presentador de datos e ilustra cómo puedes usar datos para presentar una historia convincente. Un buen lugar para empezar es este video corto filmado en conjunto con la BBC: <https://www.youtube.com/watch?v=jbkSRLYSojo>.

Los datos de gapminder resumen la progresión de países a través del tiempo, mirando estadísticos como esperanza de vida y PIB. Los datos son de fácil acceso en R gracias a Jenny Bryan, quien creó el paquete __gapminder__. Utilizaremos la versión en español contenida en el paquete __datos__, que incorpora este dataset en el objeto `paises`.


```r
library(datos)
paises
#> # A tibble: 1,704 x 6
#>   pais       continente  anio esperanza_de_vida poblacion pib_per_capita
#>   <fct>      <fct>      <int>             <dbl>     <int>          <dbl>
#> 1 Afganistán Asia        1952              28.8   8425333           779.
#> 2 Afganistán Asia        1957              30.3   9240934           821.
#> 3 Afganistán Asia        1962              32.0  10267083           853.
#> 4 Afganistán Asia        1967              34.0  11537966           836.
#> 5 Afganistán Asia        1972              36.1  13079460           740.
#> 6 Afganistán Asia        1977              38.4  14880372           786.
#> # … with 1,698 more rows
```

En este caso de estudio, nos enfocaremos solo en tres variables para responder la pregunta "¿Cómo la esperanza de vida (`esperanza_de_vida`) cambia a través del tiempo (`anio`) para cada país (`pais`)?". Un buen lugar para empezar es con un gráfico:


```r
paises %>%
  ggplot(aes(anio, esperanza_de_vida, group = pais)) +
  geom_line(alpha = 1 / 3)
```

<img src="25-model-many_files/figure-html/unnamed-chunk-2-1.png" width="70%" style="display: block; margin: auto;" />

Es un conjunto de datos pequeño: solo tiene ~1,700 observaciones y 3 variables. ¡Pero aun así es difícil ver qué está pasando! En general, pareciera que la esperanza de vida ha estado mejorando en forma constante. Sin embargo, si miras de cerca, puedes notar que algunos países no siguen este patrón. ¿Cómo podemos hacer que esos países se vean más fácilmente?

Una forma es usar el mismo enfoque que en el último capítulo: hay una señal fuerte (un crecimiento lineal general) que hace difícil ver tendencias más sutiles. Separaremos estos factores estimando un modelo con una tendencia lineal. El modelo captura el crecimiento estable en el tiempo y los residuos mostrarán lo que queda fuera.

Ya sabes cómo hacer eso si tenemos un solo país:


```r
nz <- filter(paises, pais == "Nueva Zelanda")
nz %>%
  ggplot(aes(anio, esperanza_de_vida)) +
  geom_line() +
  ggtitle("Datos completos = ")

nz_mod <- lm(esperanza_de_vida ~ anio, data = nz)
nz %>%
  add_predictions(nz_mod) %>%
  ggplot(aes(anio, pred)) +
  geom_line() +
  ggtitle("Tendencia lineal + ")

nz %>%
  add_residuals(nz_mod) %>%
  ggplot(aes(anio, resid)) +
  geom_hline(yintercept = 0, colour = "white", size = 3) +
  geom_line() +
  ggtitle("Patrón restante")
```

<img src="25-model-many_files/figure-html/unnamed-chunk-3-1.png" width="33%" /><img src="25-model-many_files/figure-html/unnamed-chunk-3-2.png" width="33%" /><img src="25-model-many_files/figure-html/unnamed-chunk-3-3.png" width="33%" />

¿Cómo podemos ajustar fácilmente ese modelo para cada país?

### Datos anidados

Te puedes imaginar copiando y pegando ese código múltiples veces; sin embargo, ¡ya has aprendido una mejor forma! Extrae el código en común con una función y repítelo usando una función map del paquete __purrr__. Este problema se estructura un poco diferente respecto a lo que has visto antes. En lugar de repetir una acción por cada variable, bnhgqueremos repetirla para cada país, es decir, un subconjunto de filas. Para hacer esto, necesitamos una nueva estructura de datos: el __*data frame* anidado__ (_nested data frame_). Para crear un _data frame_ anidado empezamos con un _data frame_ agrupado, y lo "anidamos":


```r
por_pais <- paises %>%
  group_by(pais, continente) %>%
  nest()

por_pais
#> # A tibble: 142 x 3
#> # Groups:   pais, continente [142]
#>   pais       continente data             
#>   <fct>      <fct>      <list>           
#> 1 Afganistán Asia       <tibble [12 × 4]>
#> 2 Albania    Europa     <tibble [12 × 4]>
#> 3 Argelia    África     <tibble [12 × 4]>
#> 4 Angola     África     <tibble [12 × 4]>
#> 5 Argentina  Américas   <tibble [12 × 4]>
#> 6 Australia  Oceanía    <tibble [12 × 4]>
#> # … with 136 more rows
```

(Estamos haciendo un poco de trampa agrupando por `continente` y `pais` al mismo tiempo. Dado el `pais`, `continente` es fijo, así que no agrega ningún grupo más, pero es una forma fácil de llevarnos una variable adicional para el camino.)

Esto crea un _data frame_ que tiene una fila por grupo (por país), y una columna bastante inusual: `data`. `data` es una lista de _data frames_ (o _tibbles_, para ser precisos). Esto parece una idea un poco loca: ¡tenemos un _data frame_ con una columna que es una lista de otros _data frames_! Explicaré brevemente por qué pienso que es una buena idea.

La columna `data` es un poco difícil de examinar porque es una lista moderadamente complicada y todavía estamos trabajando para tener buenas herramientas para explorar estos objetos. Desafortunadamente, usar `str()` no es recomendable porque usualmente producirá un _output_ (salida de código) muy extenso. Pero si extraes un solo elemento de la columna `data` verás que contiene todos los datos para ese país (en este caso, Afganistán).


```r
por_pais$data[[1]]
#> # A tibble: 12 x 4
#>    anio esperanza_de_vida poblacion pib_per_capita
#>   <int>             <dbl>     <int>          <dbl>
#> 1  1952              28.8   8425333           779.
#> 2  1957              30.3   9240934           821.
#> 3  1962              32.0  10267083           853.
#> 4  1967              34.0  11537966           836.
#> 5  1972              36.1  13079460           740.
#> 6  1977              38.4  14880372           786.
#> # … with 6 more rows
```

Nota la diferencia entre un _data frame_ agrupado estándar y un _data frame_ anidado: en un _data frame_ agrupado cada fila es una observación; en un _data frame_ anidado, cada fila es un grupo. Otra forma de pensar en un conjunto de datos anidado es que ahora tenemos una meta-observación: una fila que representa todo el transcurso de tiempo para un país, en lugar de solo un punto en el tiempo.

### Columnas-lista

Ahora que tenemos nuestro _data frame_ anidado, estamos en una buena posición para ajustar algunos modelos. Tenemos una función para ajustar modelos:


```r
modelo_pais <- function(df) {
  lm(esperanza_de_vida ~ anio, data = df)
}
```

Y queremos aplicarlo a cada _data frame_. Los _data frames_ están en una lista, así que podemos usar `purrr::map()` para aplicar `modelo_pais` a cada elemento:


```r
modelos <- map(por_pais$data, modelo_pais)
```

Sin embargo, en lugar de dejar la lista de modelos como un objeto suelto, lo mejor sería almacenarlo como una columna en el _data frame_ `por_pais`. Almacenar objetos relacionados en columnas es una parte clave del valor de los _data frames_, y por eso creemos que las columnas-lista son tan buena idea. En el transcurso de nuestro trabajo con estos países vamos a tener muchas listas en las que tenemos un elemento por país. ¿Por qué no almacenarlos todos juntos en un _data frame_?

En otras palabras, en lugar de crear un nuevo objeto en el entorno global, vamos a crear una nueva variable en el _data frame_ `por_pais`. Ese es un trabajo para `dplyr::mutate()`:


```r
por_pais <- por_pais %>%
  mutate(modelo = map(data, modelo_pais))
por_pais
#> # A tibble: 142 x 4
#> # Groups:   pais, continente [142]
#>   pais       continente data              modelo
#>   <fct>      <fct>      <list>            <list>
#> 1 Afganistán Asia       <tibble [12 × 4]> <lm>  
#> 2 Albania    Europa     <tibble [12 × 4]> <lm>  
#> 3 Argelia    África     <tibble [12 × 4]> <lm>  
#> 4 Angola     África     <tibble [12 × 4]> <lm>  
#> 5 Argentina  Américas   <tibble [12 × 4]> <lm>  
#> 6 Australia  Oceanía    <tibble [12 × 4]> <lm>  
#> # … with 136 more rows
```

Esto tiene una gran ventaja: como todos los objetos relacionados están almacenados juntos, no necesitas manualmente mantenerlos sincronizados cuando filtras o reordenas. La semántica del _data frame_ se ocupa de esto por ti:


```r
por_pais %>%
  filter(continente == "Europa")
#> # A tibble: 30 x 4
#> # Groups:   pais, continente [30]
#>   pais                 continente data              modelo
#>   <fct>                <fct>      <list>            <list>
#> 1 Albania              Europa     <tibble [12 × 4]> <lm>  
#> 2 Austria              Europa     <tibble [12 × 4]> <lm>  
#> 3 Bélgica              Europa     <tibble [12 × 4]> <lm>  
#> 4 Bosnia y Herzegovina Europa     <tibble [12 × 4]> <lm>  
#> 5 Bulgaria             Europa     <tibble [12 × 4]> <lm>  
#> 6 Croacia              Europa     <tibble [12 × 4]> <lm>  
#> # … with 24 more rows
por_pais %>%
  arrange(continente, pais)
#> # A tibble: 142 x 4
#> # Groups:   pais, continente [142]
#>   pais         continente data              modelo
#>   <fct>        <fct>      <list>            <list>
#> 1 Argelia      África     <tibble [12 × 4]> <lm>  
#> 2 Angola       África     <tibble [12 × 4]> <lm>  
#> 3 Benin        África     <tibble [12 × 4]> <lm>  
#> 4 Botswana     África     <tibble [12 × 4]> <lm>  
#> 5 Burkina Faso África     <tibble [12 × 4]> <lm>  
#> 6 Burundi      África     <tibble [12 × 4]> <lm>  
#> # … with 136 more rows
```

Si tu lista de _data frames_ y lista de modelos fueran objetos separados, tendrías que acordarte de que cuando reordenas o seleccionas un subconjunto de un vector, necesitas reordenar o seleccionar el subconjunto de todos los demás para mantenerlos sincronizados. Si te olvidas, tu código va a seguir funcionando, ¡pero va a devolver la respuesta equivocada!

### Desanidando

Previamente calculamos los residuos de un único modelo con un conjunto de datos también único. Ahora tenemos 142 _data frames_ y 142 modelos. Para calcular los residuos, necesitamos llamar a la función `add_residuals()` (_adicionar residuos_, en inglés) con cada par modelo-datos:


```r
por_pais <- por_pais %>%
  mutate(
    residuos = map2(data, modelo, add_residuals)
  )
por_pais
#> # A tibble: 142 x 5
#> # Groups:   pais, continente [142]
#>   pais       continente data              modelo residuos         
#>   <fct>      <fct>      <list>            <list> <list>           
#> 1 Afganistán Asia       <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> 2 Albania    Europa     <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> 3 Argelia    África     <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> 4 Angola     África     <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> 5 Argentina  Américas   <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> 6 Australia  Oceanía    <tibble [12 × 4]> <lm>   <tibble [12 × 5]>
#> # … with 136 more rows
```

¿Pero cómo puedes graficar una lista de _data frames_? En lugar de luchar para contestar esa pregunta, transformemos la lista de _data frames_ de vuelta en un _data frame_ regular. Previamente usamos `nest()` (_anidar_) para transformar un _data frame_ regular en uno anidado; ahora desanidaremos con `unnest()`:


```r
residuos <- unnest(por_pais, residuos)
residuos
#> # A tibble: 1,704 x 9
#> # Groups:   pais, continente [142]
#>   pais  continente data  modelo  anio esperanza_de_vi… poblacion pib_per_capita
#>   <fct> <fct>      <lis> <list> <int>            <dbl>     <int>          <dbl>
#> 1 Afga… Asia       <tib… <lm>    1952             28.8   8425333           779.
#> 2 Afga… Asia       <tib… <lm>    1957             30.3   9240934           821.
#> 3 Afga… Asia       <tib… <lm>    1962             32.0  10267083           853.
#> 4 Afga… Asia       <tib… <lm>    1967             34.0  11537966           836.
#> 5 Afga… Asia       <tib… <lm>    1972             36.1  13079460           740.
#> 6 Afga… Asia       <tib… <lm>    1977             38.4  14880372           786.
#> # … with 1,698 more rows, and 1 more variable: resid <dbl>
```

Nota que cada columna regular está repetida una vez por cada fila en la columna anidada.

Ahora que tenemos un _data frame_ regular, podemos graficar los residuos:


```r
residuos %>%
  ggplot(aes(anio, resid)) +
  geom_line(aes(group = pais), alpha = 1 / 3) +
  geom_smooth(se = FALSE)
#> `geom_smooth()` using method = 'gam' and formula 'y ~ s(x, bs = "cs")'
```

<img src="25-model-many_files/figure-html/unnamed-chunk-12-1.png" width="70%" style="display: block; margin: auto;" />

Separar facetas por continente es particularmente revelador:


```r
residuos %>%
  ggplot(aes(anio, resid, group = pais)) +
  geom_line(alpha = 1 / 3) +
  facet_wrap(~continente)
```

<img src="25-model-many_files/figure-html/unnamed-chunk-13-1.png" width="70%" style="display: block; margin: auto;" />

Parece que hemos perdido algunos patrones suaves. También hay algo interesante pasando en África: vemos algunos residuos muy grandes, lo que sugiere que nuestro modelo no está ajustando muy bien. Exploraremos más eso en la próxima sección, atacando el problema desde un ángulo un poco diferente.

### Calidad del modelo

En lugar de examinar los residuos del modelo, podríamos examinar algunas medidas generales de la calidad del modelo. Aprendiste cómo calcular algunas medidas específicas en el capítulo anterior. Aquí mostraremos un enfoque diferente usando el paquete __broom__. El paquete __broom__ provee un conjunto de funciones generales para transformar modelos en datos ordenados. Aquí utilizaremos `broom::glance()` (_vistazo_, en inglés) para extraer algunas métricas de la calidad del modelo. Si lo aplicamos a un modelo, obtenemos un _data frame_ con una única fila:


```r
broom::glance(nz_mod)
#> # A tibble: 1 x 11
#>   r.squared adj.r.squared sigma statistic p.value    df logLik   AIC   BIC
#>       <dbl>         <dbl> <dbl>     <dbl>   <dbl> <int>  <dbl> <dbl> <dbl>
#> 1     0.954         0.949 0.804      205. 5.41e-8     2  -13.3  32.6  34.1
#> # … with 2 more variables: deviance <dbl>, df.residual <int>
```

Podemos usar `mutate()` y `unnest()` para crear un _data frame_ con una fila por cada país:


```r
por_pais %>%
  mutate(glance = map(modelo, broom::glance)) %>%
  unnest(glance)
#> # A tibble: 142 x 16
#> # Groups:   pais, continente [142]
#>   pais  continente data  modelo residuos r.squared adj.r.squared sigma statistic
#>   <fct> <fct>      <lis> <list> <list>       <dbl>         <dbl> <dbl>     <dbl>
#> 1 Afga… Asia       <tib… <lm>   <tibble…     0.948         0.942 1.22      181. 
#> 2 Alba… Europa     <tib… <lm>   <tibble…     0.911         0.902 1.98      102. 
#> 3 Arge… África     <tib… <lm>   <tibble…     0.985         0.984 1.32      662. 
#> 4 Ango… África     <tib… <lm>   <tibble…     0.888         0.877 1.41       79.1
#> 5 Arge… Américas   <tib… <lm>   <tibble…     0.996         0.995 0.292    2246. 
#> 6 Aust… Oceanía    <tib… <lm>   <tibble…     0.980         0.978 0.621     481. 
#> # … with 136 more rows, and 7 more variables: p.value <dbl>, df <int>,
#> #   logLik <dbl>, AIC <dbl>, BIC <dbl>, deviance <dbl>, df.residual <int>
```

Este no es exactamente el _output_ que queremos, porque aún incluye todas las columnas que son una lista. Este es el comportamiento por defecto cuando `unnest()` trabaja sobre _data frames_ con una única fila. Para suprimir esas columnas usamos `.drop = TRUE` (`drop` = _descartar_, en inglés):


```r
glance <- por_pais %>%
  mutate(glance = map(modelo, broom::glance)) %>%
  unnest(glance, .drop = TRUE)
#> Warning: The `.drop` argument of `unnest()` is deprecated as of tidyr 1.0.0.
#> All list-columns are now preserved.
#> This warning is displayed once every 8 hours.
#> Call `lifecycle::last_warnings()` to see where this warning was generated.
glance
#> # A tibble: 142 x 16
#> # Groups:   pais, continente [142]
#>   pais  continente data  modelo residuos r.squared adj.r.squared sigma statistic
#>   <fct> <fct>      <lis> <list> <list>       <dbl>         <dbl> <dbl>     <dbl>
#> 1 Afga… Asia       <tib… <lm>   <tibble…     0.948         0.942 1.22      181. 
#> 2 Alba… Europa     <tib… <lm>   <tibble…     0.911         0.902 1.98      102. 
#> 3 Arge… África     <tib… <lm>   <tibble…     0.985         0.984 1.32      662. 
#> 4 Ango… África     <tib… <lm>   <tibble…     0.888         0.877 1.41       79.1
#> 5 Arge… Américas   <tib… <lm>   <tibble…     0.996         0.995 0.292    2246. 
#> 6 Aust… Oceanía    <tib… <lm>   <tibble…     0.980         0.978 0.621     481. 
#> # … with 136 more rows, and 7 more variables: p.value <dbl>, df <int>,
#> #   logLik <dbl>, AIC <dbl>, BIC <dbl>, deviance <dbl>, df.residual <int>
```

(Presta atención a las variables que no se imprimieron: hay mucha información útil allí).

Con este _data frame_ podemos empezar a buscar modelos que no se ajustan bien:


```r
glance %>%
  arrange(r.squared)
#> # A tibble: 142 x 16
#> # Groups:   pais, continente [142]
#>   pais  continente data  modelo residuos r.squared adj.r.squared sigma statistic
#>   <fct> <fct>      <lis> <list> <list>       <dbl>         <dbl> <dbl>     <dbl>
#> 1 Ruan… África     <tib… <lm>   <tibble…    0.0172      -0.0811   6.56     0.175
#> 2 Bots… África     <tib… <lm>   <tibble…    0.0340      -0.0626   6.11     0.352
#> 3 Zimb… África     <tib… <lm>   <tibble…    0.0562      -0.0381   7.21     0.596
#> 4 Zamb… África     <tib… <lm>   <tibble…    0.0598      -0.0342   4.53     0.636
#> 5 Swaz… África     <tib… <lm>   <tibble…    0.0682      -0.0250   6.64     0.732
#> 6 Leso… África     <tib… <lm>   <tibble…    0.0849      -0.00666  5.93     0.927
#> # … with 136 more rows, and 7 more variables: p.value <dbl>, df <int>,
#> #   logLik <dbl>, AIC <dbl>, BIC <dbl>, deviance <dbl>, df.residual <int>
```

Los peores modelos parecieran estar todos en África. Vamos a chequear esto con un gráfico. Tenemos un número relativamente chico de observaciones y una variable discreta, así que usar `geom_jitter()` es efectivo:


```r
glance %>%
  ggplot(aes(continente, r.squared)) +
  geom_jitter(width = 0.5)
```

<img src="25-model-many_files/figure-html/unnamed-chunk-18-1.png" width="70%" style="display: block; margin: auto;" />

Podríamos quitar los países con un $R^2$ particularmente malo y graficar los datos:


```r
mal_ajuste <- filter(glance, r.squared < 0.25)

paises %>%
  semi_join(mal_ajuste, by = "pais") %>%
  ggplot(aes(anio, esperanza_de_vida, colour = pais)) +
  geom_line()
```

<img src="25-model-many_files/figure-html/unnamed-chunk-19-1.png" width="70%" style="display: block; margin: auto;" />

Vemos dos efectos principales aquí: las tragedias de la epidemia de VIH/SIDA y el genocidio de Ruanda.

### Ejercicios

1. Una tendencia lineal parece ser demasiado simple para la tendencia general.
 ¿Puedes hacerlo mejor con un polinomio cuadrático? ¿Cómo puedes interpretar
 el coeficiente del término cuadrático? (Pista: puedes querer transformar
 `year` para que tenga media cero.)

1. Explora otros métodos para visualizar la distribución del $R^2$ por
 continente. Puedes querer probar el paquete __ggbeeswarm__, que provee
 métodos similares para evitar superposiciones como jitter, pero usa métodos
 determinísticos.

1. Para crear el último gráfico (mostrando los datos para los países con los
 peores ajustes del modelo), precisamos dos pasos: creamos un _data frame_ con
 una fila por país y después hicimos un _semi-join_ al conjunto de datos original.
 Es posible evitar este _join_ si usamos `unnest()` en lugar de
 `unnest(.drop = TRUE)`. ¿Cómo?

## Columnas-lista

Ahora que has visto un flujo de trabajo básico para manejar muchos modelos, vamos a sumergirnos en algunos detalles. En esta sección, exploraremos en más detalle la estructura de datos columna-lista. Solo recientemente hemos comenzado a apreciar realmente la idea de la columna-lista. Esta estructura está implícita en la definición de _data frame_: un _data frame_ es una lista nombrada de vectores de igual largo. Una lista es un vector, así que siempre ha sido legítimo usar una lista como una columna de un _data frame_. Sin embargo, R base no hace las cosas fáciles para crear columnas-lista, y `data.frame()` trata a la lista como una lista de columnas:


```r
data.frame(x = list(1:3, 3:5))
#>   x.1.3 x.3.5
#> 1     1     3
#> 2     2     4
#> 3     3     5
```

Puedes prevenir que `data.frame()` haga esto con `I()`, pero el resultado no se imprime particularmente bien:


```r
data.frame(
  x = I(list(1:3, 3:5)),
  y = c("1, 2", "3, 4, 5")
)
#>         x       y
#> 1 1, 2, 3    1, 2
#> 2 3, 4, 5 3, 4, 5
```

_Tibble_ mitiga este problema siendo más _perezoso_ (`tibble()` no modifica sus _inputs_) y proporcionando un mejor método de impresión:


```r
tibble(
  x = list(1:3, 3:5),
  y = c("1, 2", "3, 4, 5")
)
#> # A tibble: 2 x 2
#>   x         y      
#>   <list>    <chr>  
#> 1 <int [3]> 1, 2   
#> 2 <int [3]> 3, 4, 5
```

Es incluso más fácil con `tribble()`, ya que automáticamente puede interpretar que necesitas una lista:


```r
tribble(
  ~x, ~y,
  1:3, "1, 2",
  3:5, "3, 4, 5"
)
#> # A tibble: 2 x 2
#>   x         y      
#>   <list>    <chr>  
#> 1 <int [3]> 1, 2   
#> 2 <int [3]> 3, 4, 5
```

Las columnas-lista son usualmente más útiles como estructuras de datos intermedias. Es difícil trabajar con ellas directamente, porque la mayoría de las funciones de R trabaja con vectores atómicos o _data frames_, pero la ventaja de mantener ítems relacionados juntos en un _data frame_ hace que valga la pena un poco de molestia.

Generalmente hay tres partes en un _pipeline_ efectivo de columnas-lista:

1. Creas la columna-lista usando alguna de estas opciones: `nest()` o `summarise()` + `list()`
 o `mutate()` + una función map, como se describe en [Creando columnas-lista].

1. Creas otra columna-lista intermedia transformando columnas lista
 existentes con `map()`, `map2()` o `pmap()`. Por ejemplo,
 en el caso de estudio de arriba, creamos una columna-lista de modelos transformando
 una columna-lista de _data frames_.

1. Simplificas la columna-lista de vuelta en un _data frame_ o vector atómico,
 como se describe en [Simplificando columnas-lista].

## Creando columnas-lista

Típicamente, no tendrás que crear columnas-lista con `tibble()`, sino a partir de columnas regulares usando uno de estos tres métodos:

1. Con `tidyr::nest()` para convertir un _data frame_ agrupado en uno anidado
 en el que tengas columnas-lista de _data frames_.

1. Con `mutate()` y funciones vectorizadas que retornan una lista.

1. Con `summarise()` y funciones de resumen que retornan múltiples resultados.

Alternativamente, podrías crearlas a partir de una lista nombrada, usando `tibble::enframe()`.

Generalmente, cuando creas columnas-lista, debes asegurarte de que sean homogéneas: cada elemento debe contener el mismo tipo de cosa. No hay chequeos para asegurarte de que sea así, pero si usas __purrr__ y recuerdas lo que aprendiste sobre funciones de tipo estable (_type-stable functions_), encontrarás que eso pasa naturalmente.

### Con anidación

`nest()` crea un _data frame_ anidado, que es un _data frame_ con una columna-lista de _data frames_. En un _data frame_ anidado cada fila es una meta-observación: las otras columnas son variables que definen la observación (como país y continente arriba), y la columna-lista de _data frames_ tiene las observaciones individuales que construyen la meta-observación.

Hay dos formas de usar `nest()`. Hasta ahora has visto cómo usarlo con un _data frame_ agrupado. Cuando se aplica a un _data frame_ agrupado, `nest()` mantiene las columnas que agrupan tal cual, y envuelve todo lo demás en la columna-lista:


```r
paises %>%
  group_by(pais, continente) %>%
  nest()
#> # A tibble: 142 x 3
#> # Groups:   pais, continente [142]
#>   pais       continente data             
#>   <fct>      <fct>      <list>           
#> 1 Afganistán Asia       <tibble [12 × 4]>
#> 2 Albania    Europa     <tibble [12 × 4]>
#> 3 Argelia    África     <tibble [12 × 4]>
#> 4 Angola     África     <tibble [12 × 4]>
#> 5 Argentina  Américas   <tibble [12 × 4]>
#> 6 Australia  Oceanía    <tibble [12 × 4]>
#> # … with 136 more rows
```

También lo puedes usar en un _data frame_ no agrupado, especificando cuáles columnas quieres anidar:


```r
paises %>%
  nest(anio:pib_per_capita)
#> Warning: All elements of `...` must be named.
#> Did you want `data = c(anio, esperanza_de_vida, poblacion, pib_per_capita)`?
#> # A tibble: 142 x 3
#>   pais       continente data             
#>   <fct>      <fct>      <list>           
#> 1 Afganistán Asia       <tibble [12 × 4]>
#> 2 Albania    Europa     <tibble [12 × 4]>
#> 3 Argelia    África     <tibble [12 × 4]>
#> 4 Angola     África     <tibble [12 × 4]>
#> 5 Argentina  Américas   <tibble [12 × 4]>
#> 6 Australia  Oceanía    <tibble [12 × 4]>
#> # … with 136 more rows
```

### A partir de funciones vectorizadas

Algunas funciones útiles toman un vector atómico y retornan una lista. Por ejemplo, en [cadenas de caracteres] aprendiste sobre `stringr::str_split()`, que toma un vector de caracteres y retorna una lista de vectores de caracteres. Si lo usas dentro de `mutate`, obtendrás una columna-lista:


```r
df <- tribble(
  ~x1,
  "a,b,c",
  "d,e,f,g"
)

df %>%
  mutate(x2 = stringr::str_split(x1, ","))
#> # A tibble: 2 x 2
#>   x1      x2       
#>   <chr>   <list>   
#> 1 a,b,c   <chr [3]>
#> 2 d,e,f,g <chr [4]>
```

`unnest()` sabe cómo manejar estas listas de vectores:


```r
df %>%
  mutate(x2 = stringr::str_split(x1, ",")) %>%
  unnest()
#> Warning: `cols` is now required when using unnest().
#> Please use `cols = c(x2)`
#> # A tibble: 7 x 2
#>   x1      x2   
#>   <chr>   <chr>
#> 1 a,b,c   a    
#> 2 a,b,c   b    
#> 3 a,b,c   c    
#> 4 d,e,f,g d    
#> 5 d,e,f,g e    
#> 6 d,e,f,g f    
#> # … with 1 more row
```

(Si usas mucho este patrón, asegúrate de chequear `tidyr::separate_rows()`, que es un _wrapper_ alrededor de este patrón común).

Otro ejemplo de este patrón es usar `map()`, `map2()`, `pmap()` de __purrr__. Por ejemplo, podríamos tomar el ejemplo final de [Invocando distintas functions] y reescribirlo usando `mutate()`:


```r
sim <- tribble(
  ~f, ~params,
  "runif", list(min = -1, max = 1),
  "rnorm", list(sd = 5),
  "rpois", list(lambda = 10)
)

sim %>%
  mutate(sims = invoke_map(f, params, n = 10))
#> # A tibble: 3 x 3
#>   f     params           sims      
#>   <chr> <list>           <list>    
#> 1 runif <named list [2]> <dbl [10]>
#> 2 rnorm <named list [1]> <dbl [10]>
#> 3 rpois <named list [1]> <int [10]>
```

Nota que técnicamente `sim` no es homogéneo porque contiene tanto vectores de dobles como vectores de enteros. Sin embargo, es probable que esto no cause muchos problemas porque ambos vectores son numéricos.

### A partir de medidas de resumen con más de un valor

Una restricción de `summarise()` es que solo funciona con funciones de resumen que retornan un único valor. Eso significa que no puedes usarlo con funciones como `quantile()`, que retorna un vector de largo arbitrario:


```r
mtautos %>%
  group_by(cilindros) %>%
  summarise(q = quantile(millas))
#> `summarise()` regrouping output by 'cilindros' (override with `.groups` argument)
#> # A tibble: 15 x 2
#> # Groups:   cilindros [3]
#>   cilindros     q
#>       <dbl> <dbl>
#> 1         4  21.4
#> 2         4  22.8
#> 3         4  26  
#> 4         4  30.4
#> 5         4  33.9
#> 6         6  17.8
#> # … with 9 more rows
```

Sin embargo, ¡puedes envolver el resultado en una lista! Esto obedece el contrato de `summarise()`, porque cada resumen ahora es una lista (un vector) de largo 1.


```r
mtautos %>%
  group_by(cilindros) %>%
  summarise(q = list(quantile(millas)))
#> `summarise()` ungrouping output (override with `.groups` argument)
#> # A tibble: 3 x 2
#>   cilindros q        
#>       <dbl> <list>   
#> 1         4 <dbl [5]>
#> 2         6 <dbl [5]>
#> 3         8 <dbl [5]>
```

Para producir resultados útiles con `unnest`, también necesitarás capturar las probabilidades:


```r
probs <- c(0.01, 0.25, 0.5, 0.75, 0.99)
mtautos %>%
  group_by(cilindros) %>%
  summarise(p = list(probs), q = list(quantile(millas, probs))) %>%
  unnest()
#> `summarise()` ungrouping output (override with `.groups` argument)
#> Warning: `cols` is now required when using unnest().
#> Please use `cols = c(p, q)`
#> # A tibble: 15 x 3
#>   cilindros     p     q
#>       <dbl> <dbl> <dbl>
#> 1         4  0.01  21.4
#> 2         4  0.25  22.8
#> 3         4  0.5   26  
#> 4         4  0.75  30.4
#> 5         4  0.99  33.8
#> 6         6  0.01  17.8
#> # … with 9 more rows
```

### A partir de una lista nombrada

Los _data frames_ con columnas-lista proveen una solución a un problema común: ¿qué haces si quieres iterar sobre el contenido de una lista y también sobre sus elementos? En lugar de tratar de juntar todo en un único objeto, usualmente es más fácil hacer un _data frame_: una columna puede contener los elementos y otra columna la lista. Una forma fácil de crear un _data frame_ como este desde una lista es `tibble::enframe()`.


```r
x <- list(
  a = 1:5,
  b = 3:4,
  c = 5:6
)

df <- enframe(x)
df
#> # A tibble: 3 x 2
#>   name  value    
#>   <chr> <list>   
#> 1 a     <int [5]>
#> 2 b     <int [2]>
#> 3 c     <int [2]>
```

La ventaja de esta estructura es que se generaliza de una forma relativamente sencilla - los nombres son útiles si tienes un vector de caracteres con los metadatos, pero no ayudan para otros tipos de datos o para múltiples vectores.

Ahora, si quieres iterar sobre los nombres y valores en paralelo, puedes usar `map2()`:


```r
df %>%
  mutate(
    smry = map2_chr(name, value, ~ stringr::str_c(.x, ": ", .y[1]))
  )
#> # A tibble: 3 x 3
#>   name  value     smry 
#>   <chr> <list>    <chr>
#> 1 a     <int [5]> a: 1 
#> 2 b     <int [2]> b: 3 
#> 3 c     <int [2]> c: 5
```

### Ejercicios

1. Lista todas las funciones en las que puedas pensar que tomen como _input_ un vector atómico y
 retornen una lista.

1. Piensa en funciones de resumen útiles que, como `quantile()`, retornen
 múltiples valores.

1. ¿Qué es lo que falta en el siguiente _data frame_? ¿Cómo `quantile()` retorna
 eso que falta? ¿Por qué eso no es tan útil aquí?


```r
mtautos %>%
  group_by(cilindros) %>%
  summarise(q = list(quantile(millas))) %>%
  unnest()
#> `summarise()` ungrouping output (override with `.groups` argument)
#> Warning: `cols` is now required when using unnest().
#> Please use `cols = c(q)`
#> # A tibble: 15 x 2
#>   cilindros     q
#>       <dbl> <dbl>
#> 1         4  21.4
#> 2         4  22.8
#> 3         4  26  
#> 4         4  30.4
#> 5         4  33.9
#> 6         6  17.8
#> # … with 9 more rows
```

1. ¿Qué hace este código? ¿Por qué podría ser útil?


```r
mtautos %>%
  group_by(cilindros) %>%
  summarise_each(funs(list))
```

## Simplificando columnas-lista

Para aplicar las técnicas de manipulación de datos y visualización que has aprendido en este libro, necesitarás simplificar la columna-lista de vuelta a una columna regular (un vector atómico) o conjunto de columnas. La técnica que usarás para volver a una estructura más sencilla depende de si quieres un único valor por elemento, o múltiples valores.

1. Si quieres un único valor, usa `mutate()` con `map_lgl()`,
 `map_int()`, `map_dbl()`, y `map_chr()` para crear un vector atómico.

1. Si quieres varios valores, usa `unnest()` para convertir columnas-lista de vuelta a
 columnas regulares, repitiendo las filas tantas veces como sea necesario.

Estas técnicas están descritas con más detalle abajo.

### Lista a vector

Si puedes reducir tu columna lista a un vector atómico entonces será una columna regular. Por ejemplo, siempre puedes resumir un objeto con su tipo y largo, por lo que este código funcionará sin importar cuál tipo de columna-lista tengas:


```r
df <- tribble(
  ~x,
  letters[1:5],
  1:3,
  runif(5)
)

df %>% mutate(
  tipo = map_chr(x, typeof),
  largo = map_int(x, length)
)
#> # A tibble: 3 x 3
#>   x         tipo      largo
#>   <list>    <chr>     <int>
#> 1 <chr [5]> character     5
#> 2 <int [3]> integer       3
#> 3 <dbl [5]> double        5
```

Esta es la misma información básica que obtienes del método de impresión por defecto de _tbl_ , solo que ahora lo puedes usar para filtrar. Es una técnica útil si tienes listas heterogéneas y quieres remover las partes que no te sirven.

No te olvides de los atajos de `map_*()` - puedes usar `map_chr(x, "manzana")` para extraer la cadena de caracteres almacenada en `manzana` para cada elemento de `x`. Esto es útil para separar listas anidadas en columnas regulares. Usa el argumento `.null` para proveer un valor para usar si el elemento es un valor faltante (en lugar de retornar `NULL`):


```r
df <- tribble(
  ~x,
  list(a = 1, b = 2),
  list(a = 2, c = 4)
)
df %>% mutate(
  a = map_dbl(x, "a"),
  b = map_dbl(x, "b", .null = NA_real_)
)
#> # A tibble: 2 x 3
#>   x                    a     b
#>   <list>           <dbl> <dbl>
#> 1 <named list [2]>     1     2
#> 2 <named list [2]>     2    NA
```

### Desanidando

`unnest()` trabaja repitiendo la columna regular una vez para cada elemento de la columna-lista. Por ejemplo, en el siguiente ejemplo sencillo repetimos la primera fila 4 veces (porque el primer elemento de `y` tiene largo cuatro) y la segunda fila una vez:


```r
tibble(x = 1:2, y = list(1:4, 1)) %>% unnest(y)
#> # A tibble: 5 x 2
#>       x     y
#>   <int> <dbl>
#> 1     1     1
#> 2     1     2
#> 3     1     3
#> 4     1     4
#> 5     2     1
```

Esto significa que no puedes simultáneamente desanidar dos columnas que contengan un número diferente de elementos:


```r
# Funciona, porque y y z tienen el mismo número de elementos en
# cada fila
df1 <- tribble(
  ~x, ~y, ~z,
  1, c("a", "b"), 1:2,
  2, "c", 3
)
df1
#> # A tibble: 2 x 3
#>       x y         z        
#>   <dbl> <list>    <list>   
#> 1     1 <chr [2]> <int [2]>
#> 2     2 <chr [1]> <dbl [1]>
df1 %>% unnest(y, z)
#> Warning: unnest() has a new interface. See ?unnest for details.
#> Try `df %>% unnest(c(y, z))`, with `mutate()` if needed
#> # A tibble: 3 x 3
#>       x y         z
#>   <dbl> <chr> <dbl>
#> 1     1 a         1
#> 2     1 b         2
#> 3     2 c         3

# No funciona porque y y z tienen un número diferente de elementos
df2 <- tribble(
  ~x, ~y, ~z,
  1, "a", 1:2,
  2, c("b", "c"), 3
)
df2
#> # A tibble: 2 x 3
#>       x y         z        
#>   <dbl> <list>    <list>   
#> 1     1 <chr [1]> <int [2]>
#> 2     2 <chr [2]> <dbl [1]>
df2 %>% unnest(y, z)
#> Warning: unnest() has a new interface. See ?unnest for details.
#> Try `df %>% unnest(c(y, z))`, with `mutate()` if needed
#> # A tibble: 4 x 3
#>       x y         z
#>   <dbl> <chr> <dbl>
#> 1     1 a         1
#> 2     1 a         2
#> 3     2 b         3
#> 4     2 c         3
```

El mismo principio aplica al desanidar columnas-lista de _data frames_. Puedes desanidar múltiples columnas-lista siempre que todos los _data frames_ de cada fila tengan la misma cantidad de filas.

### Ejercicios

1. ¿Por qué podría ser útil la función `lengths()` para crear columnas de
 vectores atómicos a partir de columnas-lista?

1. Lista los tipos de vectores más comúnes que se encuentran en un _data frame_. ¿Qué hace que las
 listas sean diferentes?

## Haciendo datos ordenados con broom

El paquete __broom__ provee tres herramientas generales para transformar modelos en _data frames_ ordenados:

1. `broom::glance(modelo)` retorna una fila para cada modelo. Cada columna tiene una
 medida de resumen del modelo: o bien una medida de la calidad del modelo, o bien complejidad, o
 una combinación de ambos.

1. `broom::tidy(modelo)` retorna una fila por cada coeficiente en el modelo. Cada
 columna brinda información acerca de la estimación o su variabilidad.

1. `broom::augment(modelo, data)` retorna una fila por cada fila en `data`, agregando
 valores adicionales como residuos, y estadísticos de influencia.
