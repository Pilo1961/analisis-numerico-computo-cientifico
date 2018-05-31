---
title: Reporte: Singular Value Decomposition on GPU using CUDA.
author: Alejandro Daniel Pérez Palacios, 167256
date: 31/Mayo/2018
---
# Reporte: Singular Value Decomposition on GPU using CUDA.

La factorización SVD es una técnica importante utilizada para la factorización de una matriz rectangular. Bajo SVD las
operaciones con matrices son más robustas por lo que se utiliza para calcular la pesudo-inversa de una matriz, resolver sistemas de ecuaciones lineales u optimización de mínimos cuadrados, entre otros.

En principio la factorización SVD de una matriz **A** está definida por:
<a href="http://www.codecogs.com/eqnedit.php?latex=A=U&space;\Sigma&space;V^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?A=U&space;\Sigma&space;V^T" title="A=U \Sigma V^T" /></a>
donde <a href="http://www.codecogs.com/eqnedit.php?latex=U&space;\in&space;\mathbb{R}^{mxn},&space;V&space;\in&space;\mathbb{R}^{nxn},&space;\Sigma&space;\in&space;\mathbb{R}^{mxn}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?U&space;\in&space;\mathbb{R}^{mxn},&space;V&space;\in&space;\mathbb{R}^{nxn},&space;\Sigma&space;\in&space;\mathbb{R}^{mxn}" title="U \in \mathbb{R}^{mxn}, V \in \mathbb{R}^{nxn}, \Sigma \in \mathbb{R}^{mxn}" /></a> matrices
ortognales y <a href="http://www.codecogs.com/eqnedit.php?latex=\Sigma" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\Sigma" title="\Sigma" /></a> matriz diagonal con <a href="http://www.codecogs.com/eqnedit.php?latex=s_{ii}>=0" target="_blank"><img src="http://latex.codecogs.com/gif.latex?s_{ii}>=0" title="s_{ii}>=0" /></a> con orden descendiente.

El objetivo del artículo _Singular Value Decomposotion on GPU using CUDA_ es realizar esta factorización en la GPU aprovechando el increíble poder de cómputo que la caracteriza. El rápido crecimiento en desempeño del hardware de gráficos llevó a la GPU a ser un fuerte candidato para realizar múltiples operaciones de caracter intensivo y en paralelo. Es por esto que han surgido diversos ambientes que soporten la programación en GPU, NVIDIA provee CUDA como un ambiente operacional
con un lenguaje tipo C para los procesadores programables.

Este artículo expone que aunque el cómputo en la GPU se ha aprovechado para el cómputo científico no lo ha sido para la factorización SVD, problema que por sus múltiples aplicaciones se ve necesario realizar dicha implementación.

La factorización SVD se puede obtener por diversos algortimos, en este artículo se expone el de Golub-Reinsch (Bidiagonalización y diagonalización) dado que es simple y compacto además de mapear la arquitectura SIMD a la GPU. En resumen este algortimo realiza lo siguiente:

  1. La matriz A se reduce a una matriz bidiagonal usando una serie de transformaciones HouseHolder.
  2. La matriz reducida se diagonaliza utilizando iteraciones QR alternadas implícitamente.

En notación matemática se resume de la siguiente manera:

  1. <a href="http://www.codecogs.com/eqnedit.php?latex=B&space;\leftarrow&space;Q^TAP" target="_blank"><img src="http://latex.codecogs.com/gif.latex?B&space;\leftarrow&space;Q^TAP" title="B \leftarrow Q^TAP" /></a> {Bidiagonalización de A a B}
  2. <a href="http://www.codecogs.com/eqnedit.php?latex=\Sigma&space;\leftarrow&space;X^TBY" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\Sigma&space;\leftarrow&space;X^TBY" title="\Sigma \leftarrow X^TBY" /></a> {Diagonalización de B a <a href="http://www.codecogs.com/eqnedit.php?latex=\Sigma" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\Sigma" title="\Sigma" /></a>}
  3. <a href="http://www.codecogs.com/eqnedit.php?latex=U&space;\leftarrow&space;QX" target="_blank"><img src="http://latex.codecogs.com/gif.latex?U&space;\leftarrow&space;QX" title="U \leftarrow QX" /></a>
  4. <a href="http://www.codecogs.com/eqnedit.php?latex=V^T&space;\leftarrow&space;(PY)^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?V^T&space;\leftarrow&space;(PY)^T" title="V^T \leftarrow (PY)^T" /></a> {Calcular matrices U,V^T ortogonales y SVD de <a href="http://www.codecogs.com/eqnedit.php?latex=A=U&space;\Sigma&space;V^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?A=U&space;\Sigma&space;V^T" title="A=U \Sigma V^T" /></a>}

Es aquí cuando comienza lo interesante, debemos encontrar un algoritmo para poder realizar la bidiagonalización de A a B; para este caso en particular se realiza una serie de transformaciones Householder unitarias que nos ayudarán a obtener una matriz con ceros en la i-ésima fila e i-ésima columna, por ejemplo para la primera iteración: Para una matriz A de mxn, tomamos <a href="http://www.codecogs.com/eqnedit.php?latex=u^{(1)}\in\mathbb{R}^m" target="_blank"><img src="http://latex.codecogs.com/gif.latex?u^{(1)}\in\mathbb{R}^m" title="u^{(1)}\in\mathbb{R}^m" /></a> para el vector A(1:m,1) y <a href="http://www.codecogs.com/eqnedit.php?latex=v^{(1)}\in\mathbb{R}^n" target="_blank"><img src="http://latex.codecogs.com/gif.latex?v^{(1)}\in\mathbb{R}^n" title="v^{(1)}\in\mathbb{R}^n" /></a> para A(1,2:n)
tal que:

<a href="http://www.codecogs.com/eqnedit.php?latex=\begin{align*}&space;\hat{A}&space;&=&space;(I-\sigma_{1,1}u^{(1)}{u^{(1)}}^T)\\&space;&=&space;H_1AG_1&space;=\begin{bmatrix}&space;\alpha_1&space;&&space;\beta_1&&space;0&&space;\cdots&&space;0\\&space;0&&space;x&&space;x&&space;\cdots&&space;x\\&space;\vdots&&space;x&&space;x&&space;\cdots&&space;x\\&space;0&&space;x&&space;x&&space;\cdots&&space;x&space;\end{bmatrix}&space;\end{align*}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\begin{align*}&space;\hat{A}&space;&=&space;(I-\sigma_{1,1}u^{(1)}{u^{(1)}}^T)\\&space;&=&space;H_1AG_1&space;=\begin{bmatrix}&space;\alpha_1&space;&&space;\beta_1&&space;0&&space;\cdots&&space;0\\&space;0&&space;x&&space;x&&space;\cdots&&space;x\\&space;\vdots&&space;x&&space;x&&space;\cdots&&space;x\\&space;0&&space;x&&space;x&&space;\cdots&&space;x&space;\end{bmatrix}&space;\end{align*}" title="\begin{align*} \hat{A} &= (I-\sigma_{1,1}u^{(1)}{u^{(1)}}^T)\\ &= H_1AG_1 =\begin{bmatrix} \alpha_1 & \beta_1& 0& \cdots& 0\\ 0& x& x& \cdots& x\\ \vdots& x& x& \cdots& x\\ 0& x& x& \cdots& x \end{bmatrix} \end{align*}" /></a>

De esta manera eliminamos la primer columna y la primera fila y debemos realizar más iteraciones para obtener la matriz bidiagonal. Las matrices izquierdas <a href="http://www.codecogs.com/eqnedit.php?latex=H_i's" target="_blank"><img src="http://latex.codecogs.com/gif.latex?H_i's" title="H_i's" /></a> y las derechas <a href="http://www.codecogs.com/eqnedit.php?latex=G_i's" target="_blank"><img src="http://latex.codecogs.com/gif.latex?G_i's" title="G_i's" /></a> son llamadas matrices Householder, la eliminación siguiente se obtiene:

<a href="http://www.codecogs.com/eqnedit.php?latex=\begin{align*}&space;B&space;&=&space;Q^TAP&space;\end{align*}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\begin{align*}&space;B&space;&=&space;Q^TAP&space;\end{align*}" title="\begin{align*} B &= Q^TAP \end{align*}" /></a>

donde,

<a href="http://www.codecogs.com/eqnedit.php?latex=\begin{align*}&space;Q^t&space;&=&space;\prod_{i=1}^nH_iP=\prod_{i=1}^{n-2}G_i&space;\end{align*}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\begin{align*}&space;Q^t&space;&=&space;\prod_{i=1}^nH_iP=\prod_{i=1}^{n-2}G_i&space;\end{align*}" title="\begin{align*} Q^t &= \prod_{i=1}^nH_iP=\prod_{i=1}^{n-2}G_i \end{align*}" /></a>

aquí <a href="http://www.codecogs.com/eqnedit.php?latex=H_i=I-\sigma_{1,i}u^{(i)}{u^{(i)}}^t" target="_blank"><img src="http://latex.codecogs.com/gif.latex?H_i=I-\sigma_{1,i}u^{(i)}{u^{(i)}}^t" title="H_i=I-\sigma_{1,i}u^{(i)}{u^{(i)}}^t" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=G_i=I-\sigma_{2,i}v^{(i)}{v^{(i)}}^t" target="_blank"><img src="http://latex.codecogs.com/gif.latex?G_i=I-\sigma_{2,i}v^{(i)}{v^{(i)}}^t" title="G_i=I-\sigma_{2,i}v^{(i)}{v^{(i)}}^t" /></a>, los vectores u y v son una elección de vectores householder para la transformación. En general para un vector <a href="http://www.codecogs.com/eqnedit.php?latex=y&space;\in&space;\mathbb{R}^l" target="_blank"><img src="http://latex.codecogs.com/gif.latex?y&space;\in&space;\mathbb{R}^l" title="y \in \mathbb{R}^l" /></a> la selección de vectores householder **r** y escalares <a href="http://www.codecogs.com/eqnedit.php?latex=\sigma" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\sigma" title="\sigma" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=\alpha" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\alpha" title="\alpha" /></a> está dado por lo siguiente:

<a href="http://www.codecogs.com/eqnedit.php?latex=\begin{align*}&space;\alpha&space;&=&space;-signo(y_1)||y||,&space;a=sign(y_1)||y||\\&space;\sigma&space;&=&space;(y_1&plus;a)/a\\&space;\mathbf{r}&space;&=&space;\frac{y&plus;[a,0,\cdots,0]^T}{y_1&plus;a}&space;\end{align*}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\begin{align*}&space;\alpha&space;&=&space;-signo(y_1)||y||,&space;a=sign(y_1)||y||\\&space;\sigma&space;&=&space;(y_1&plus;a)/a\\&space;\mathbf{r}&space;&=&space;\frac{y&plus;[a,0,\cdots,0]^T}{y_1&plus;a}&space;\end{align*}" title="\begin{align*} \alpha &= -signo(y_1)||y||, a=sign(y_1)||y||\\ \sigma &= (y_1+a)/a\\ \mathbf{r} &= \frac{y+[a,0,\cdots,0]^T}{y_1+a} \end{align*}" /></a>

tal que <a href="http://www.codecogs.com/eqnedit.php?latex=(I-\sigma\mathbf{rr}^T)y=[\alpha,0,...,0]^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?(I-\sigma\mathbf{rr}^T)y=[\alpha,0,...,0]^T" title="(I-\sigma\mathbf{rr}^T)y=[\alpha,0,...,0]^T" /></a>. En nuestro caso tenemos que los vectores u y v serán calculados como **r** y los escalares <a href="http://www.codecogs.com/eqnedit.php?latex=\alpha_i" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\alpha_i" title="\alpha_i" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=\beta_i" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\beta_i" title="\beta_i" /></a> serán calculados como <a href="http://www.codecogs.com/eqnedit.php?latex=\alpha" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\alpha" title="\alpha" /></a>.

De esta manera obtenemos los vectores para realizar las transformaciones y poder realizar con éxito las eliminaciones de cada renglón-columna (hacerlos ceros). Podemos entonces resumir la actualización de la matriz A después de la i-ésima eliminación renglón-columna de la siguiente manera:

<a href="http://www.codecogs.com/eqnedit.php?latex=A(i&plus;1:m,i&plus;1:n)=A(i&plus;1:m,i&plus;1:n)-\hat{\mathbf{u}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T-\hat{\mathbf{w}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?A(i&plus;1:m,i&plus;1:n)=A(i&plus;1:m,i&plus;1:n)-\hat{\mathbf{u}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T-\hat{\mathbf{w}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T" title="A(i+1:m,i+1:n)=A(i+1:m,i+1:n)-\hat{\mathbf{u}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T-\hat{\mathbf{w}}^{(i)}{\hat{\mathbf{z}}^{(i)}}^T" /></a>

donde, 

<a href="http://www.codecogs.com/eqnedit.php?latex=\begin{align*}&space;(\hat{\mathbf{z}}^{(i)})^T&space;&=&space;\mathbf{x}^T-\sigma_{2,i}(\mathbf{x}^T\hat{\mathbf{v}}^{(i)})(\hat{\mathbf{v}}^{(i)})^T\\&space;\hat{\mathbf{w}}^{(i)}&=&space;\sigma_{2,i}A(i:m,i&plus;1:n)\hat{\mathbf{v}}^{(i)}\\&space;\mathbf{x}&=&space;\sigma_{1,i}A^T(i:m,i&plus;1:n)\hat{\mathbf{u}}^{(i)}&space;\end{align*}" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\begin{align*}&space;(\hat{\mathbf{z}}^{(i)})^T&space;&=&space;\mathbf{x}^T-\sigma_{2,i}(\mathbf{x}^T\hat{\mathbf{v}}^{(i)})(\hat{\mathbf{v}}^{(i)})^T\\&space;\hat{\mathbf{w}}^{(i)}&=&space;\sigma_{2,i}A(i:m,i&plus;1:n)\hat{\mathbf{v}}^{(i)}\\&space;\mathbf{x}&=&space;\sigma_{1,i}A^T(i:m,i&plus;1:n)\hat{\mathbf{u}}^{(i)}&space;\end{align*}" title="\begin{align*} (\hat{\mathbf{z}}^{(i)})^T &= \mathbf{x}^T-\sigma_{2,i}(\mathbf{x}^T\hat{\mathbf{v}}^{(i)})(\hat{\mathbf{v}}^{(i)})^T\\ \hat{\mathbf{w}}^{(i)}&= \sigma_{2,i}A(i:m,i+1:n)\hat{\mathbf{v}}^{(i)}\\ \mathbf{x}&= \sigma_{1,i}A^T(i:m,i+1:n)\hat{\mathbf{u}}^{(i)} \end{align*}" /></a>

Las matrices <a href="http://www.codecogs.com/eqnedit.php?latex=P" target="_blank"><img src="http://latex.codecogs.com/gif.latex?P" title="P" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=Q" target="_blank"><img src="http://latex.codecogs.com/gif.latex?Q" title="Q" /></a> de la bidiagonalización <a href="http://www.codecogs.com/eqnedit.php?latex=B=Q^TAP" target="_blank"><img src="http://latex.codecogs.com/gif.latex?B=Q^TAP" title="B=Q^TAP" /></a> son obtenidas de una forma similar. PAra obtener la matriz <a href="http://www.codecogs.com/eqnedit.php?latex=B" target="_blank"><img src="http://latex.codecogs.com/gif.latex?B" title="B" /></a> se utilizó bidiagonalización parcial debido a que es menos costoso y fue previamente implementado en GPU.


Una vez descrita la bidiagonalización se menciona que las actualizaciones se pueden expresar utilizando operaciones de nivel 2 de BLAS, actualizando la matriz cada que ocurre una eliminación renglón-columna. Dado que el método es computacionalmente costoso se propone dividir la matriz en bloques y a cada bloque aplicar la bidiagonalización, si dividimos a la matriz en L bloques entonces la actualización completa de la matriz se dará cuando L columnas y renglones sean bidiagonalizados. Dado que cada bloque sufre una actualización se requieren cálculos extra de los vectores u,v,w y z. Dado que cada bloque tendrá sus vectores householder es necesario almacenar en memoria los cada uno en una matriz U,V,W y Z respectivamente ya que estos serán necesarios para la actualización de la matriz completa.

El algortimo descrito es implementado en GPU mediante CUBLAS, el enfoque por bloques tiene un desempeño alto debido a que las funciones de multiplicación y normas que tiene CUBLAS para matriz-matriz y matriz-vector. Se menciona que CUBLAS se desempeña mejor para matrices cuyo tamaño es un múltiplo de 32 (debido a que se alineaba con la capacidad de memoria) y se aprovechó este hecho al agregar ceros a las matrices y vectores para que su longitud fuer aun múltiplo de 32.

Algo que es importante mencionar es que el desempeño de la GPU también depende de en donde se encuentren alojados los datos. Se asume que inicialmente los datos(la matriz) se encuentran en la CPU y son transferidos a la GPU. Además dado que el bando de ancha entre la CPU y GPU es de magnitud menor al de sólo GPU se propone inicializar las matrices invloucradas en la bidiagonalización en ls GPU. Una vez que se ha bidiagonalizado la matriz A esta es copiada a la CPU para proceder con la diagonalización y se mantienen la matrices <a href="http://www.codecogs.com/eqnedit.php?latex=Q" target="_blank"><img src="http://latex.codecogs.com/gif.latex?Q" title="Q" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=P^T" target="_blank"><img src="http://latex.codecogs.com/gif.latex?P^T" title="P^T" /></a> en la GPU.

Después de realizar la bidiagonalización se obtiene la matriz <a href="http://www.codecogs.com/eqnedit.php?latex=B" target="_blank"><img src="http://latex.codecogs.com/gif.latex?B" title="B" /></a>, la cual queremos ahora diagonalizar de tal forma que obtengamos <a href="http://www.codecogs.com/eqnedit.php?latex=\Sigma=X^TBY" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\Sigma=X^TBY" title="\Sigma=X^TBY" /></a> donde <a href="http://www.codecogs.com/eqnedit.php?latex=\Sigma" target="_blank"><img src="http://latex.codecogs.com/gif.latex?\Sigma" title="\Sigma" /></a> es una matriz diagonal y X,Y^T son matrices ortogonales. 

Como se mencionó al principio, se realizará la diagonalización con el algoritmo _Implicitly shifted QR_, para esto se definen  <a href="http://www.codecogs.com/eqnedit.php?latex=d(i)" target="_blank"><img src="http://latex.codecogs.com/gif.latex?d(i)" title="d(i)" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=e(i)" target="_blank"><img src="http://latex.codecogs.com/gif.latex?e(i)" title="e(i)" /></a> como la i-ésima entrada de la diagonal y diagonal superior respectivamente, de tal manera que en cada iteración del algoritmo la diagonal superior vaya haciendose más pequeña. Al converger, <a href="http://www.codecogs.com/eqnedit.php?latex=d(i)'s" target="_blank"><img src="http://latex.codecogs.com/gif.latex?d(i)'s" title="d(i)'s" /></a> serán los valores singulares y <a href="http://www.codecogs.com/eqnedit.php?latex=X^T,&space;Y" target="_blank"><img src="http://latex.codecogs.com/gif.latex?X^T,&space;Y" title="X^T, Y" /></a> contendrán los vectores singulares de <a href="http://www.codecogs.com/eqnedit.php?latex=B" target="_blank"><img src="http://latex.codecogs.com/gif.latex?B" title="B" /></a>. 

El algortimo descrito en el artículo encuentra los índices <a href="http://www.codecogs.com/eqnedit.php?latex=k_1" target="_blank"><img src="http://latex.codecogs.com/gif.latex?k_1" title="k_1" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=k_2" target="_blank"><img src="http://latex.codecogs.com/gif.latex?k_2" title="k_2" /></a> para los cuales <a href="http://www.codecogs.com/eqnedit.php?latex=e(k_1)" target="_blank"><img src="http://latex.codecogs.com/gif.latex?e(k_1)" title="e(k_1)" /></a> sea menor a un límite que depende de la precisión de la computadora. Si los índices <a href="http://www.codecogs.com/eqnedit.php?latex=k_1" target="_blank"><img src="http://latex.codecogs.com/gif.latex?k_1" title="k_1" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=k_2" target="_blank"><img src="http://latex.codecogs.com/gif.latex?k_2" title="k_2" /></a> difieren por 1 o 2 se pueden extraer directamente los valors singulares, en otro caso se realiza una serie de rotaciones de <a href="http://www.codecogs.com/eqnedit.php?latex=d(i)" target="_blank"><img src="http://latex.codecogs.com/gif.latex?d(i)" title="d(i)" /></a> y <a href="http://www.codecogs.com/eqnedit.php?latex=e(i)" target="_blank"><img src="http://latex.codecogs.com/gif.latex?e(i)" title="e(i)" /></a> de tal manera que las <a href="http://www.codecogs.com/eqnedit.php?latex=e(i)'s" target="_blank"><img src="http://latex.codecogs.com/gif.latex?e(i)'s" title="e(i)'s" /></a> sean menores que la iteración su valor en la iteración anterior, los cálculos y rotaciones se detienen hasta encontrar todos los valores singulares.

En cuanto a la implementación de este algoritmo en la GPU se hizo una combinación de calculos en paralelo y secuencial para obtener el resultado final. Primero se copian las diagonales y superdiagonales de la matriz B a la CPU, en esta se realizan las rotaciones de manera secuencial ya que sólo se requiere acceso a las diagonales. Los algortimos para realizar las rotaciones dependen de las filas anteriores y por lo tanto es difícil paralelizarlo, aún asi lo que se paralelizó son los calculos de las operaciones dentro de cada fila resultando en gran deseempeño desde matrices medianas hasta matrices grandes.

Una vez que se han obtenido las matrices Q,X y P^T, Y^T basta con realizar las multiplicaciones para obtener U=QX y V^T=(PY)^T.

## Resultados

Para evaluar el desempeño del algoritmo se comparó contra las implementaciones en matlab y LAPACK, las cuales ya han sido optimizadas y son el mejor punto de comparación. Además se probó la implementación en GPU con diversos equipos y distintas capacidades de memoria además de tarjetas gráficas.

Es interesante cómo los autores evaluaron el resultado, ya no sólo lo hicieron en distintos equipos sino que para cada equipo se realizó el cálculo de SVD generando matrices aleatorias y se corrió el algortimo 10 veces para cada matriz. Además se promediaron los tiempos de cómputo de las 10 matrices para tomar en cuenta que la muestra pueda ser muy buena o muy mala.  En los resultdos mostrados es a partir de matrices de dimensión 512 que se nota la gran diferencia de la implementación en GPU, obtiendo un speed-up 0.61 sobre la implementación en INTEL MKL(el algoritmo óptimo en CPU) y con un speed-up de 7.8 con respecto al mismo algoritmo pero para una matriz de 4Kx4K. 

El speed-up se realiza con respecto al algoritmo de MKL ya que este también involucra una bidiagonalización parcial de la matriz. Dicho lo anterior se comparó cada uno de los pasos para calcular la descomposición SVD, es decir se calculó el speed-up con respecto a MKL de la bidiagonalización, diagonalización y multiplicación final. Esto demostró que la parte del proceso con el mayor speed-up para matrices de dimensión grande fue en la bidiagonalización parcial y en algunos casos superado por la diagonalización ya que su crecimiento en speed-up tiende a aumentar gradualmente con el tamaño de las matrices.

El mayor logro es el cálculo de SVD para matrices de gran tamaño (desde 14Kx14K), que son prácticamente imposibles de manejar en la CPU por el almacenamiento en memoria. Para estas matrices el cálculo de la SVD tómo un total de 76 minutos en la GPU con el algoritmo descrito (GPU NVIDIA Tesla S1070).

Un resultado, bastante importante, es el hecho de generar un algoritmo híbrido para la descomposición ya que no depende únicamente de la GPU, reconociendo las limitaciones y más que nada aprovechando el poder de computo que tiene la CPU para operaciones más sencillas en secuencial.


