# Práctica 3: Estimación no paramétrica {#practica3}





En esta práctica nos centraremos en el bootstrap
en la estimación tipo núcleo de la densidad y de la función de regresión,
para la aproximación de la precisión y el sesgo,
y también para el cálculo de intervalos de confianza y de predicción
(en futuras versiones de los apuntes se incluirá también 
detalles sobre la construcción de bandas de confianza).

## Estimación no paramétrica de la densidad

Como ya se comentó en la Sección \@ref(cap4-boot-suav),
en `R` podemos emplear la función `density()` del paquete base para obtener
una estimación tipo núcleo de la densidad. 
Los principales parámetros (con los valores por defecto) son los siguientes:
```
density(x, bw = "nrd0", adjust = 1, kernel = "gaussian", n = 512, from, to)
```

- `bw`: ventana, puede ser un valor numérico o una cadena de texto que la determine
  (en ese caso llamará internamente a la función `bw.xxx()` donde `xxx` se corresponde
  con la cadena de texto). Las opciones son:

    - `"nrd0"`, `"nrd"`: Reglas del pulgar de Silverman (1986, page 48, eqn (3.31)) y 
      Scott (1992), respectivamente. Como es de esperar que la densidad objetivo 
      no sea tan suave como la normal, estos criterios tenderán a seleccionar 
      ventanas que producen un sobresuavizado de las observaciones.

    - `"ucv"`, `"bcv"`: Métodos de validación cruzada insesgada y sesgada, respectivamente.
    
    - `"sj"`, `"sj-ste"`, `"sj-dpi"`: Métodos de Sheather y Jones (1991), 
        "solve-the-equation" y "direct plug-in", respectivamente.
 
-   `adjust`: parameto para reescalado de la ventana, las estimaciones se calculan 
    con la ventana `adjust*bw`.

-   `kernel`: cadena de texto que determina la función núcleo, las opciones son: `"gaussian"`,
    `"epanechnikov"`, `"rectangular"`, `"triangular"`, `"biweight"`, `"cosine"` y `"optcosine"`.
    
-   `n`, `from`, `to`: permiten establecer la rejilla en la que se obtendrán las estimaciones
    (si $n>512$ se emplea `fft()` por lo que se recomienda establecer `n` a un múltiplo de 2;
    por defecto `from` y `to` se establecen como `cut = 3` veces la ventana desde los extremos 
    de las observaciones).

Utilizaremos como punto de partida el código empleado en la Sección \@ref(cap4-boot-suav).
Considerando el conjunto de datos `precip` (que contiene el promedio de precipitación, 
en pulgadas de lluvia, de 70 ciudades de Estados Unidos).


```r
x <- precip
h <- bw.SJ(x)
npden <- density(x, bw = h)
# npden <- density(x, bw = "SJ")

# plot(npden)
hist(x, freq = FALSE, main = "Kernel density estimation",
     xlab = paste("Bandwidth =", formatC(h)), lty = 2,
     border = "darkgray", xlim = c(0, 80), ylim = c(0, 0.04))
lines(npden, lwd = 2)
rug(x, col = "darkgray")
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-2-1} \end{center}

Alternativamente podríamos emplear implementaciones en otros paquetes de `R`.
Uno de los más empleados es `ks` (Duong, 2019), que admite estimación 
incondicional y condicional multidimensional.
También se podrían emplear los paquetes `KernSmooth` (Wand y Ripley, 2019), 
`sm` (Bowman y Azzalini, 2019), `np` (Tristen y Jeffrey, 2019), 
`kedd` (Guidoum, 2019), `features` (Duong y Matt, 2019) y `npsp` (Fernández-Casal, 2019), 
entre otros.

## Bootstrap y estimación no paramétrica de la densidad

La idea sería aproximar la distribución del error de estimación 
$\hat f(x) - f(x)$ por la distribución bootstrap de
$\hat f^{\ast}(x) - \hat f(x)$ (bootstrap percentil básico).

Como se comentó en la Sección \@ref(aproximacion-bootstrap) 
la ventana $g$ ha de ser asintóticamente mayor que $h$ (de orden $n^{-1/5}$) 
y la recomendación sería emplear la ventana óptima para la estimación de 
$f^{\prime \prime }\left( x \right)$, de orden $n^{-1/9}$. 


```r
# Remuestreo
set.seed(1)
n <- length(x)
g <- h * n^(4/45) # h*n^(-1/9)/n^(-1/5)
range.x <- range(npden$x) # Para fijar las posiciones de estimación
B <- 1000
stat_den_boot <- matrix(nrow = length(npden$x), ncol = B)
for (k in 1:B) {
    # x_boot <- sample(x, n, replace = TRUE) + rnorm(n, 0, g)
    x_boot <- rnorm(n, sample(x, n, replace = TRUE), g)
    den_boot <- density(x_boot, bw = h, from = range.x[1], to = range.x[2])$y
    # Si se quiere tener en cuenta la variabilidad debida a la selección de
    # la ventana habría que emplear el mismo criterio en la función `density`.
    stat_den_boot[, k] <- den_boot - npden$y
}

# Calculo del sesgo y error estándar 
bias <- apply(stat_den_boot, 1, mean)
std.err <- apply(stat_den_boot, 1, sd)

# Representar estimación y corrección de sesgo bootstrap
plot(npden, type="l", ylim = c(0, 0.05), lwd = 2)
lines(npden$x, npden$y - bias)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-3-1} \end{center}

```r
# lines(npden$x, pmax(0, npden$y - bias))
```


### Intervalos de confianza

Empleando la aproximación descrita en la Sección \@ref(cap5-basic)
podemos cálcular de estimaciones por intervalo de confianza (puntuales)
por el método percentil (básico).


```r
alfa <- 0.05
pto_crit <- apply(stat_den_boot, 1, quantile, probs = c(alfa/2, 1 - alfa/2))
# ic_inf_boot <- npden$y - pto_crit[2, ]
ic_inf_boot <- pmax(0, npden$y - pto_crit[2, ])
ic_sup_boot <- npden$y - pto_crit[1, ]

plot(npden, type="l", ylim = c(0, 0.05), lwd = 2)
lines(npden$x, pmax(0, npden$y - bias))
lines(npden$x, ic_inf_boot, lty = 2)
lines(npden$x, ic_sup_boot, lty = 2)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-4-1} \end{center}


### Implementación con el paquete `boot`

Como también se comentó en la Sección \@ref(cap4-boot-suav), 
la recomendación es implementar el bootstrap suavizado como un bootstrap paramétrico:


```r
library(boot)

# Los objetos necesarios para el cálculo del estadístico
# hay que pasarlos a traves del argumento `data` de `boot`.
range.x <- range(npden$x)
data.precip <- list(x = x, h = h, range.x = range.x)

ran.gen.smooth <- function(data, mle) {
    # Función para generar muestras aleatorias mediante
    # bootstrap suavizado con función núcleo gaussiana,
    # mle contendrá la ventana
    n <- length(data$x)
    g <- mle
    xboot <- rnorm(n, sample(data$x, n, replace = TRUE), g)
    out <- list(x = xboot, h = data$h, range.x = data$range.x)
}

statistic <- function(data) 
                density(data$x, bw = data$h, from = range.x[1], to = range.x[2])$y

set.seed(1)
res.boot <- boot(data.precip, statistic, R = B, sim = "parametric",
                 ran.gen = ran.gen.smooth, mle = g)

# Calculo del sesgo y error estándar
bias <- with(res.boot, apply(t, 2, mean, na.rm = TRUE) -  t0)
std.err <- apply(res.boot$t, 2, sd, na.rm = TRUE)
```

Además, la función `boot.ci()` solo permite el cálculo del intervalo de 
confianza para cada valor de $x$ de forma independiente (parámetro `index`). 
Por lo que podría ser recomendable obtenerlo a partir de las réplicas 
bootstrap del estimador:


```r
# Método percentil básico calculado directamente 
# a partir de las réplicas bootstrap del estimador
alfa <- 0.05
pto_crit <- apply(res.boot$t, 2, quantile, probs = c(alfa/2, 1 - alfa/2))
ic_inf_boot <- pmax(0, 2*npden$y - pto_crit[2, ])
ic_sup_boot <- 2*npden$y - pto_crit[1, ]

plot(npden, ylim = c(0, 0.05), lwd = 2)
lines(npden$x, pmax(0, npden$y - bias))

lines(npden$x, ic_inf_boot, lty = 2)
lines(npden$x, ic_sup_boot, lty = 2)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-6-1} \end{center}

En la práctica, en muchas ocasiones se trabaja directamente con
las réplicas bootstrap del estimador. Por ejemplo, es habitual
generar envolventes como medida de la precisión de la estimación
(que se interpretan de forma similar a una banda de confianza):

```r
matplot(npden$x, t(res.boot$t), type = "l", col = "darkgray")
lines(npden, lwd = 2)
lines(npden$x, npden$y - bias)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-7-1} \end{center}

Pero la recomendación es emplear bootstrap básico (o percentil-*t*) en lugar
de bootstrap percentil (directo) en la presencia de sesgo:

```r
matplot(npden$x, 2*npden$y - t(res.boot$t), type = "l", col = "darkgray")
lines(npden, lwd = 2)
lines(npden$x, npden$y - bias)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-8-1} \end{center}


## Estimación no paramétrica de la regresión

Suponemos el modelo general: 
$$Y(\mathbf{x})=\mu(\mathbf{x})+\varepsilon(\mathbf{x}),$$ 
donde $\mu(\mathbf{x})$ es la función de regresión o tendencia y 
$\varepsilon(\mathbf{x})$ es un error aleatorio de media cero y varianza 
$\sigma^2(\mathbf{x})$, que normalmente se supone constante.

<!-- NOTACIÓN: $\mu$ v.s. $m$ en apuntes -->

NOTA: El modelo anterior se corresponde con el denominado *diseño fijo*.
Alternativamente se podría pensar que el valor de la variable explicativa
es aleatorio, *diseño aleatorio*. 
En cuyo caso el modelo anterior reflejaría la distribución condicional 
$\left. Y\right\vert_{X=x}$, siendo $\mu(x) = E\left( \left. Y\right\vert_{X=x} \right)$
y $\sigma^2(\mathbf{x}) = Var\left( \left. Y\right\vert_{X=x} \right)$


El estimador de Nadaraya-Watson de $\mu(\mathbf{x})$ descrito en el Capítulo \@ref(cap7)
es un caso particular de una clase más amplia de estimadores no paramétricos, 
denominados estimadores polinómicos locales. 


###  Regresión polinómica local

En el caso univariante, para cada $x_0$ se ajusta un polinomio:
$$\beta_0+\beta_{1}\left(x - x_0\right) + \cdots 
+ \beta_{p}\left( x-x_0\right)^{p}$$ 
por mínimos cuadrados ponderados, con pesos
$w_{i} = \frac{1}{h}K\left(\frac{x-x_0}{h}\right)$. 

-   La estimación en $x_0$ es $\hat{\mu}_{h}(x_0)=\hat{\beta}_0$.

-   Adicionalmente^[Se puede pensar que se están estimando los coeficientes de 
    un desarrollo de Taylor de $\mu(x_0)$.]: 
    $\widehat{\mu_{h}^{(r)}}(x_0) = r!\hat{\beta}_{r}$.

Habitualmente se considera:

-   $p=0$: Estimador Nadaraya-Watson.

-   $p=1$: Estimador lineal local.

Asintóticamente el estimador lineal local tiene un sesgo menor que el de 
Nadaraya-Watson (pero del mismo orden) y la misma varianza (e.g. Fan and Gijbels, 1996).
Sin embargo, su principal ventaja es que se ve menos afectado por el denominado
efecto frontera (*edge effect*).

El caso multivariante es análogo. La estimación lineal local multivariante
$\hat{\mu}_{\mathbf{H}}(\mathbf{x})=\hat{\beta}_0$ se obtiene al minimizar:
$$\begin{aligned}
    \min_{\beta_0 ,\boldsymbol{\beta}_1, \ldots, \boldsymbol{\beta}_p}
    \sum_{i=1}^{n}\left\{ Y(\mathbf{x}_{i})-\beta_0 
    -\boldsymbol{\beta}_1^t(\mathbf{x}_{i}-\mathbf{x}) - \ldots \right. \nonumber \\
    \left. -\boldsymbol{\beta}_p^t(\mathbf{x}_{i}-\mathbf{x})^p \right\}^{2}
    K_{\mathbf{H}}(\mathbf{x}_{i}-\mathbf{x}),
\end{aligned}$$ 
donde:

-   $\mathbf{H}$ es la matriz de ventanas $d\times d$ (simétrica no singular).

-   $K_{\mathbf{H}}(\mathbf{u})=\left\vert \mathbf{H}\right\vert
    ^{-1}K(\mathbf{H}^{-1}\mathbf{u})$ y $K$ núcleo multivariante.

Explícitamente:
$$\hat{\mu}_{\mathbf{H}}(\mathbf{x}) = \mathbf{e}_{1}^t \left(
X_{\mathbf{x}}^t {W}_{\mathbf{x}} 
X_{\mathbf{x}} \right)^{-1} X_{\mathbf{x}}^t 
{W}_{\mathbf{x}}\mathbf{Y} \equiv {s}_{\mathbf{x}}^t\mathbf{Y},$$
donde $\mathbf{e}_{1} = \left( 1, \cdots, 0\right)^t$, $X_{\mathbf{x}}$ 
es la matriz con $(1,(\mathbf{x}_{i}-\mathbf{x})^t, \ldots, (\mathbf{x}_{i}-{\mathbf{x})^p}^t)$ en fila $i$,
y $W_{\mathbf{x}} = \mathtt{diag} \left( 
K_{\mathbf{H}}(\mathbf{x}_{1} - \mathbf{x}), \ldots,
K_{\mathbf{H}}(\mathbf{x}_{n}-\mathbf{x}) \right)$
es la matriz de pesos.

Se puede pensar que se obtiene aplicando un suavizado polinómico a 
$(\mathbf{x}_i, Y(\mathbf{x}_i))$:
$$\hat{\boldsymbol{\mu}} = S\mathbf{Y},$$ 
siendo $S$ la matriz de suavizado con $\mathbf{s}_{\mathbf{x}_{i}}^t$ en la fila $i$.

Aunque el paquete base de `R` incluye herramientas para la estimación
tipo núcleo de la regresión (`lowess()`, `ksmooth()`), recomiendan
el uso del paquete `KernSmooth` (Wand y Ripley, 2019). 
Otros paquetes incluyen más funcionalidades: `sm` (Bowman y Azzalini, 2019), 
`np` (Tristen y Jeffrey, 2019), `npsp` (Fernández-Casal, 2019), entre otros.

Como ejemplo emplearemos el conjunto de datos `MASS::mcycle` que contiene mediciones 
de la aceleración de la cabeza en una simulación de un accidente de motocicleta, 
utilizado para probar cascos protectores.


```r
data(mcycle, package = "MASS")
x <- mcycle$times
y <- mcycle$accel  

library(KernSmooth)
h <- dpill(x, y) # Método plug-in de Ruppert, Sheather y Wand (1995)
fit <- locpoly(x, y, bandwidth = h) # Estimación lineal local
plot(x, y)
lines(fit)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-9-1} \end{center}


### Estimación de la varianza

En el caso heterocedástico, se puede obtener una estimación de la varianza 
$\sigma^2(\mathbf{x})$ mediante suavizado local de los residuos al cuadrado
(Fan y Yao, 1998). Mientras que en el caso homocedástico, se puede obtener 
una estimación de la varianza a partir de la suma de cuadrados residual y la 
matriz de suavizado:
$$\hat\sigma^2 = \frac{RSS}{df_e},$$
siendo $RSS=\Sigma_{i=1}^n \left( Y(\mathbf{x}_i) - \hat\mu(\mathbf{x}_i) \right)^2$
y $df_e = tr(I - S)$ (de forma análoga al caso lineal), o alternativamente $df_e = tr \left( (I - S^t)(I - S)\right)$, es una aproximación de los grados de libertad del error.

Adicionalmente:
$$\widehat{Var}\left(\hat{\mu}_{\mathbf{H}}(\mathbf{x}_i)\right) = \hat\sigma^2\sum_{j=1}^n s^2_{ij}$$

Uno de los pocos paquetes de `R` que implementan la estimación de la varianza
y el cálculo de intervalos de confianza es el paquete^[El paquete `np` calcula 
estimaciones similares aunque no documenta la aproximación que emplea. También 
implementa bootstrap uniforme y bootstrap por bloques.] `sm` (Bowman y Azzalini, 2019).


```r
library(sm)
hcv <- hcv(x, y) # Método de validación cruzada
fit.sm <- sm.regression(x, y, h = hcv, display = "se")
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-10-1} \end{center}

```r
fit.sm$sigma
```

```
## [1] 22.82508
```

Alternativamente se podría emplear bootstrap.

## Bootstrap residual

El modelo ajustado de regresión se puede emplear para estimar la respuesta media 
$\mu(\mathbf{x}_0)$ cuando las variables explicativas toman un valor concreto $\mathbf{x}_0$.
En este caso también podemos emplear el bootstrap residual (o el Wild Bootstrap
descrito más adelante) para realizar inferencias acerca de la media.
La idea sería aproximar la distribución del error de estimación 
$\hat\mu(\mathbf{x}_0) - \mu(\mathbf{x}_0)$ por la distribución bootstrap de
$\hat\mu^{\ast}(\mathbf{x}_0) - \hat\mu(\mathbf{x}_0)$.

Hay que tener en cuenta que el paquete `KernSmooth` no implementa los métodos
`predict()` y `residuals()`:

```r
est <- approx(fit, xout = x)$y # est <- predict(fit)
resid <- y - est # resid <- residuals(fit)
```

NOTA: De forma análoga al caso lineal, se podrían reescalar los residuos 
a partir de la matriz de suavizado (empleando los paquetes `sm` o `npsp`).

Para reproducir adecuadamente el sesgo del estimador, la ventana $g$ ha de ser asintóticamente mayor que $h$ (de orden $n^{-1/5}$).
Análogamente al caso de la densidad, la recomendación es emplear la ventana 
óptima para la estimación de $\mu^{\prime \prime }\left( \mathbf{x}_0 \right)$, 
de orden $n^{-1/9}$ (Sección \@ref(wild-bootstrap)). 

```r
n <- length(x)
g <- h * n^(4/45) # h*n^(-1/9)/n^(-1/5)
fit2 <- locpoly(x, y, bandwidth = g)
est2 <- approx(fit2, xout = x)$y # est2 <- predict(fit2)
# resid2 <- y - est2 # resid2 <- residuals(fit2)

# Remuestreo
set.seed(1)
B <- 1000
stat_fit_boot <- matrix(nrow = length(fit$x), ncol = B)
resid0 <- resid - mean(resid)
for (k in 1:B) {
    y_boot <- est2 + sample(resid0, replace = TRUE)
    fit_boot <- locpoly(x, y_boot, bandwidth = h)$y
    stat_fit_boot[ , k] <- fit_boot - fit$y
}

# Calculo del sesgo y error estándar 
bias <- apply(stat_fit_boot, 1, mean)
std.err <- apply(stat_fit_boot, 1, sd)

# Representar estimación y corrección de sesgo bootstrap
plot(x, y)
lines(fit, lwd = 2)
lines(fit$x, fit$y - bias)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-12-1} \end{center}


### Intervalos de confianza y predicción

De forma análoga a la estimación de la densidad, podemos cálcular de estimaciones 
por intervalo de confianza (puntuales) por el método percentil (básico):

```r
alfa <- 0.05
pto_crit <- apply(stat_fit_boot, 1, quantile, probs = c(alfa/2, 1 - alfa/2))
ic_inf_boot <- fit$y - pto_crit[2, ]
ic_sup_boot <- fit$y - pto_crit[1, ]

plot(x, y)
lines(fit, lwd = 2)
lines(fit$x, fit$y - bias)
lines(fit$x, ic_inf_boot, lty = 2)
lines(fit$x, ic_sup_boot, lty = 2)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-13-1} \end{center}


El modelo ajustado también es empleado para predecir una nueva respuesta individual 
$Y(\mathbf{x}_0)$ para un valor concreto $\mathbf{x}_0$ de las variables explicativas. 
En el caso de errores independientes $\hat{Y}(\mathbf{x}_0) = \hat\mu(\mathbf{x}_0)$,
pero si estamos interesados en realizar inferencias sobre el error de predicción 
$r(\mathbf{x}_0) = Y(\mathbf{x}_0) - \hat{Y}(\mathbf{x}_0)$, a la variabilidad
de $\hat\mu(\mathbf{x}_0)$ debida a la muestra, se añade la variabilidad del error 
$\varepsilon(\mathbf{x}_0)$.

La idea sería aproximar la distribución del error de predicción:
$$r(\mathbf{x}_0) = Y(\mathbf{x}_0) - \hat{Y}(\mathbf{x}_0)
= \mu(\mathbf{x}_0) + \varepsilon(\mathbf{x}_0) - \hat\mu(\mathbf{x}_0)$$
por la distribución bootstrap de:
$$r^{\ast}(\mathbf{x}_0) = Y^{\ast}(\mathbf{x}_0) - \hat{Y}^{\ast}(\mathbf{x}_0)
= \hat\mu(\mathbf{x}_0) + \varepsilon^{\ast}(\mathbf{x}_0) - \hat\mu^{\ast}(\mathbf{x}_0)$$


```r
# Remuestreo
set.seed(1)
n_pre <- length(fit$x)
stat_pred_boot <- matrix(nrow = n_pre, ncol = B)
for (k in 1:B) {
    y_boot <- est2 + sample(resid0, replace = TRUE)
    fit_boot <- locpoly(x, y_boot, bandwidth = h)$y
    pred_boot <- fit_boot + sample(resid0, n_pre, replace = TRUE)
    stat_pred_boot[ , k] <- pred_boot - fit_boot
}

# Cálculo de intervalos de predicción
# por el método percentil (básico)
alfa <- 0.05
pto_crit_pred <- apply(stat_pred_boot, 1, quantile, probs = c(alfa/2, 1 - alfa/2))
ip_inf_boot <- fit$y + pto_crit_pred[1, ]
ip_sup_boot <- fit$y + pto_crit_pred[2, ]

plot(x, y, ylim = c(-150, 75))
lines(fit, lwd = 2)
lines(fit$x, fit$y - bias)
lines(fit$x, ic_inf_boot, lty = 2)
lines(fit$x, ic_sup_boot, lty = 2)
lines(fit$x, ip_inf_boot, lty = 3)
lines(fit$x, ip_sup_boot, lty = 3)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-14-1} \end{center}

En este caso puede no ser recomendable considerar errores i.i.d.,
sería de esperar heterocedásticidad (e incluso dependencia temporal).
El bootstrap residual se puede extender al caso heterocedástico 
y/o dependencia (e.g. Castillo-Páez et al., 2019a, 2019b).

## Wild bootstrap {#p3-wild-bootstrap}

Como se describe en la Sección \@ref(wild-bootstrap)
en el caso heterocedástico se puede emplear wild bootstrap.


```r
# Remuestreo
set.seed(1)
B <- 1000
fit_boot <- matrix(nrow = n_pre, ncol = B)
for (k in 1:B) {
		rwild <- sample(c((1 - sqrt(5))/2, (1 + sqrt(5))/2), n, replace = TRUE, 
		                prob = c((5 + sqrt(5))/10, 1 - (5 + sqrt(5))/10))
    y_boot <- est2 + resid*rwild
    fit_boot[ , k] <- locpoly(x, y_boot, bandwidth = h)$y
    # OJO: bootstrap percetil directo
}
		

# Calculo del sesgo y error estándar
bias <- apply(fit_boot, 1, mean, na.rm = TRUE) -  fit$y
std.err <- apply(fit_boot, 1, sd, na.rm = TRUE)

# Representar estimación y corrección de sesgo bootstrap
plot(x, y)
lines(fit, lwd = 2)
lines(fit$x, fit$y - bias)
```



\begin{center}\includegraphics[width=0.7\linewidth]{24-Practica_3_files/figure-latex/unnamed-chunk-15-1} \end{center}


## Ejercicio (para entregar)

Siguiendo con el conjunto de datos `MASS::mcycle`, emplear wild bootstrap para 
obtener estimaciones por intervalo de confianza de la función de regresión
de `accel` a partir de `times` mediante bootstrap percentil básico.
Comparar los resultados con los obtenidos mediante bootstrap residual (comentar).


## Bibliografía

*   Bock M., Bowman A.W. y Ismail B. (2007). Estimation and inference for 
    error variance in bivariate nonparametric regression. *Statistics and Computing*, 
    **17**, 39-47.
    
*   Bowman A.W. y Azzalini A. (2019). R package 'sm': nonparametric smoothing methods 
    (version 2.2). <http://www.stats.gla.ac.uk/~adrian/sm>.    
    
*   Castillo-Páez S., Fernández-Casal R. y García-Soidán P. (2019a).
    [A nonparametric bootstrap method for spatial data ](https://www.sciencedirect.com/science/article/pii/S0167947319300325?via%3Dihub),
    *Computational Statistics and Data Analysis*, **137**, 1-15. 
  
*   Castillo-Páez S., Fernández-Casal R. y García-Soidán P. (2019b).
    [Nonparametric bootstrap approach for unconditional risk mapping under heteroscedasticity ](https://doi.org/10.1016/j.spasta.2019.100389). 
    *Spatial Statistics*. Aceptado para publicación.    

*   Fan J. y Gijbels I. (1996). *Local Polynomial Modelling and Its Applications*.
    Chapman and Hall, London.
    
*   Fan J. y Yao Q. (1998). Efficient estimation of conditional variance functions in 
    stochastic regression. *Biometrika*, **85**, 645–660.
    
*   Rupert D. and Wand M.P. (1994) Multivariate locally weighted least squares regression. 
    *The Annals of Statistics*, **22**, 1346-1370.

*   Wand M.P. y Jones M.C. (1995) *Kernel Smoothing*. Chapman and Hall, London.

