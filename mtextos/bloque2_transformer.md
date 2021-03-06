
La arquitectura transformer
===========================

En 2017 las redes neuronales recurrentes basadas en unidades LSTM como las que hemos estudiado eran la arquitectura habitual para el procesamiento neuronal de secuencias, en general, y del lenguaje natural, en particular. Algunos investigadores comenzaban a obtener también buenos resultados en esta área con las redes neuronales convolucionales, tradicionalmente empleadas con imágenes. Por otro lado, los mecanismos de atención introducidos unos años antes en las redes recurrentes habían mejorado su capacidad para resolver ciertas tareas y abierto el abánico de posibilidades de estos modelos. Además, el modelo conocido como codificador-descodificador (*encoder-decoder* en inglés) se convertía en la piedra angular de los sistemas que transformaban una secuencia en otra (sistemas conocidos como *seq2seq* como, por ejemplo, los sistemas de traducción automática o de obtención de resúmenes). A mediados de 2017, sin embargo, aparece un artículo {cite}`allyouneed` que propone eliminar la recurrencia del modelo codificador-descodificador y sustituirla por lo que se denomina autoatención (*self-attention*); aunque el artículo se centra en la tarea de la traducción automática, en muy poco tiempo la aplicación de esta arquitectura, bautizada como *transformer*, en muchos otros campos se descubre altamente eficaz hasta el punto de relegar a las arquitecturas recurrentes a un segundo plano. El transformer sería, además, uno de los elementos fundamentales de los modelos preentrenados que estudiaremos más adelante y que comenzarían a aparecer en los meses o años siguientes.


## El transformer

Como primera aproximación, puedes usar la guía ilustrada de Jay Alammar sobre el [transformer][transformer].

[transformer]: http://jalammar.github.io/illustrated-transformer/

Los contenidos nucleares del tema son los incluidos en las secciones 11.1 ("Bidirectional transformer encoders") y 9.7 ("Self-attention networks: transformers") de la tercera edición del libro "[Speech and language processing][jurafskybook]". Además, la arquitectura completa codificador-descodificador del transformer se presenta en el apartado 10.6. La aplicación del transformer a algunas tareas de procesamiento del lenguaje natural se ve en 9.8 y 9.9. Los modelos preentrenados se estudian en los apartados 11.2 y 11.3. La estrategia de búsqueda *beam search* y la tokenización en subpalabras se exploran en 10.5 y 2.4.3, respectivamente. Los elementos más básicos, comunes a cualquier tipo de arquitectura neuronal, se estudian en el capítulo 7. Aunque las hemos estudiado más por encima, el capítulo 9 se centra en las arquitecturas recurrentes. Usa todos esos contenidos junto con los apuntes de clase en tu aprendizaje. 

[jurafskybook]: https://web.stanford.edu/~jurafsky/slp3/

```{admonition} Nota
:class: note
Las representaciones aprendidas tras el entrenamiento por un transformer en cada una de sus capas para una nueva frase de entrada pueden considerarse (de la misma manera que con una red recurrente) como embeddings contextuales de los diferentes tokens de la entrada que pueden usarse a la hora de representarlos en otras tareas. En principio, cualquier capa puede ser adecuada para obtener estas representaciones, pero algunos trabajos han demostrado que ciertas capas son más adecuadas que otras para ciertas tareas. Las capas más cercanas a la entrada parecen representar información más relacionada con la morfología, mientras que las capas finales se relacionan más con la semántica.
```

### Ecuaciones del transformer

El transformer usa la arquitectura codificador-descodificador para emitir de forma autoregresiva una secuencia de salida $\boldsymbol{y}= y_1, y_2,\ldots,y_n$ a partir de una secuencia de entrada $\boldsymbol{x}= x_1, x_2,\ldots,x_n$. Habitualmente cada $x_i$ será un embedding *no contextual* para el token correspondiente de la frase a procesar obtenido de una tabla de embeddings, y cada $y_i$ será el vector de probabilidades correspondiente al $i$-ésimo token de la frase de salida.

El codificador tiene $N$ capas idénticas, cada una formada a su vez por dos subcapas:

$$
\underline{\boldsymbol{h}}^l &=& \text{LN}\left(\text{SelfAtt}\left(\boldsymbol{h}^{l-1}\right) + \boldsymbol{h}^{l-1}\right) \\[2ex]
\boldsymbol{h}^l &=& \text{LN}\left(\text{FF}\left(\underline{\boldsymbol{h}}^l\right) + \underline{\boldsymbol{h}}^l\right)
$$

donde $\boldsymbol{h}^l = \{h_1^l,h_2^l,\ldots,h_n^l\}$ son las salidas de la capa $l$-ésima (una por cada token de la entrada). La salida de la primera capa es $\boldsymbol{h}^0= \boldsymbol{x}$. La función LN obtiene la normalización a nivel de capa, SelfAtt es el mecanismo de atención con múltiples cabezales y FF es una red hacia delante completamente conectada.

El descodificador sigue un planteamiento similar con un par de particularidades: el mecanismo de autoatención usa una máscara para no usar los embeddings de los tokens aún no generados y aparece una tercera subcapa responsable de la atención hacia el codificador:

$$
\underline{\boldsymbol{s}}^l &=& \text{LN}\left(\text{MaskedSelfAtt}\left(\boldsymbol{s}^{l-1}\right) + \boldsymbol{s}^{l-1}\right) \\[2ex]
\underline{\underline{\boldsymbol{s}}}^l &=& \text{LN}\left(\text{CrossAtt}\left(\underline{\boldsymbol{s}}^{l},\boldsymbol{h}^N\right) + \underline{\boldsymbol{s}}^{l}\right) \\[2ex]
\boldsymbol{s}^l &=& \text{LN}\left(\text{FF}\left(\underline{\underline{\boldsymbol{s}}}^l\right) + \underline{\underline{\boldsymbol{s}}}^l\right)
$$

donde $\boldsymbol{s}^l$ son las salidas de la capa $l$-ésima del descodificador. Los embeddings de la última capa $\boldsymbol{s}^M$ se pasan por una capa densa adicional seguida de una función softmax para obtener la estimación de la probabilidad del token correspondiente. La salida de la primera capa del descodificador $\boldsymbol{s}^0$ es, como en el codificador, un embedding no contextual del token anterior (por ejemplo, el token de mayor probabilidad emitido en el paso anterior).

```{figure} images/self-attention_multihead-romain-futrzynski.svg
---
height: 700px
name: fig-selfatt
---
Una representación tridimensional del mecanismo de autoatención tomada del [tutorial][tutorialromain] de Romain Futrzynski.
```

[tutorialromain]: https://peltarion.com/blog/data-science/self-attention-video


### Un símil del mecanismo de autoatención

El mecanismo de autoatención se puede introducir con propósitos didácticos basándonos en una hipotética versión de Python en la que se permitiera acceder a los valores de un diccionario usando claves *aproximadas*. Supongamos el siguiente diccionario de Python almacenado en la variable `d`; como cualquier diccionario de Python este contiene también un conjunto de claves (`manzana`, por ejemplo) y sus valores asociados (`8` es el valor asociado a la clave `manzana`, por ejemplo):

```python
d = {"manzana":8, "albaricoque":4, "naranja":3}
```

En Python *convencional* ahora podemos realizar una *consulta* al diccionario con una sintaxis como `d["manzana"]` para obtener el valor `8`. El intérprete de Python ha usado el nombre de nuestra consulta (`manzana`) para buscar entre todas las claves del diccionario una cuyo nombre coincida *exactamente* y devolver su valor (`8` en este caso).

Observa cómo en la discusión anterior hemos usado los términos "consulta" (*query*), "clave" (*key*) y "valor" (*value*) que aparecen tambien cuando se discute el mencanismo de autoatención del transformer.

Vayamos ahora más allá y consideremos que realizamos una consulta como `d["narancoque"]`. Un intérprete de Python *real* lanzará una excepción ante la consulta anterior, pero un intérprete *imaginario* podría recorrer el diccionario, comparar el término de la consulta con cada clave del diccionario y ponderar los valores en función del parecido encontrado. Consideremos una función `similitud` que recibe dos cadenas y devuelve un número, no necesariamente acotado, que es mayor cuanto más parecidas son las cadenas (los valores concretos no son ahora relevantes):

```
similitud("narancoque","manzana") → 0
similitud("narancoque","albaricoque") → 20
similitud("narancoque","naranja") → 30
```

Estos resultados normalizados para que su suma sea 1 son `0`, `0,4` y `0,6`. Nuestro intérprete de Python imaginario podría ahora devolvernos para la consulta `d["narancoque"]` el valor 0 x 8 + 0,4 x 4 + 0,6 x 3 = 3,4. 

En el caso del transformer, las consultas, las claves y los valores son vectores de una cierta dimensión, y la función de similitud empleada es el producto escalar de la consulta y las diferentes claves. Los grados de similitud se normalizan mediante la función softmax y se utlizan igualmente para ponderar después los distintos valores:

$$
\text{SelfAtt}(Q,K,V) = \text{softmax}\left( \frac{Q K^T}{\sqrt{d_k}} \right) V
$$

### Normalización de capa

Sean $\hat{\mu}$ y $\hat{\sigma}^2$ la media y la varianza, respectivamente, de todas las entradas, que representaremos por $\boldsymbol{x}$, a las neuronas de una capa formada por H neuronas:

$$
\hat{\mu} &=& \frac{1}{H} \sum_{i=1}^H x_i \\[1.5ex]
\hat{\sigma}^2 &=& \frac{1}{H} \sum_{i=1}^H \left(x_i - \hat{\mu} \right)^2 + \epsilon
$$

donde $\epsilon$ tiene un valor muy pequeño para evitar que la una división por cero en la siguiente ecuación. La función LN de normalización para cada entrada de la capa se define como la estandarización:

$$
\text{LN}(x_i) = \gamma_i \frac{x_i - \hat{\mu}}{\hat{\sigma}^2} + \beta
$$

La fracción permite que todos las entradas de la capa en un determinado instante tengan media cero y varianza 1. Como estos valores son arbitrarios, en cualquier caso, se añaden dos parámetros aprendibles $\boldsymbol{\gamma}$ y $\boldsymbol{\beta}$ para reescalarlos. Los valores normalizados se convierten en la nueva entrada de cada neurona y a estos se aplica la función de activación que corresponda; en el caso del transformer, no hay ninguna función de activación adicional.

## Visualización de embeddings contextuales

Mediante la herramienta [exBERT][exbert] vamos a explorar visualmente las representaciones intermedias obtenidas en el codificador de un transformer. 

[exbert]: https://huggingface.co/exbert/?model=bart-large&modelKind=bidirectional&sentence=The%20moon%20is%20shinning%20brightly%20tonight.

En la [herramienta] puedes seleccionar el modelo a utilizar (en estos momentos no hemos estudiado las diferencias entre ellas, por lo que con *BART* es suficiente), la frase de entrada, el grado de la atención, las capas y los cabezales a mostrar. La capa superior (la más alejada de la entrada) está a la izquierda. Para seleccionar o deseleccionar un cabezal puedes hacer clic en las columnas. Puedes seleccionar un token haciendo clic sobre él y ocultarlo con doble clic. Al colocarte sobre una palabra puedes ver la predicción que haría el modelo del token que corresponde al embedding obtenido por la red en esa posición, capa y cabezal; observa que si ocultas *moon* a la derecha, por ejemplo, a la izquierda el sistema tiende a predecir *sun* en esa posición.

[herramienta]: https://www.youtube.com/watch?v=e31oyfo_thY


## Código en Pytorch del transformer

El documento "[The Annotated Transformer]" nos ayuda a entender mejor el modelo al ir mostrando paso a paso el código de Pytorch que lo implementa. En clase nos centraremos en la parte 1 ("Model architecture"), pero puedes profundizar un poco más llegando hasta la parte 2 ("Model training"), que incluye un ejemplo completo de entrenamiento para una tarea sencilla. En el ficjero `README` del repositorio de Github puedes encontrar un enlace a un *notebook* en Google Colab.

[The Annotated Transformer]: https://github.com/harvardnlp/annotated-transformer/


## Referencias

```{bibliography} bloque2_transformer.bib
```
