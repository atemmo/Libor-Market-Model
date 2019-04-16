
# Libor Market Models

## Introduction

Financial derivatives market constitutes a big portion of the financial market, and among them, interest rate derivatives play a pivotal role. In addition to its huge size, this market is also famous for its complexity. There are many types of rate, like federal fund rate, libor, treasury rate, and each rate has its own term structure, which consists the information of its forward rates. For each rate, there could be multiple products, from most basic ones like FRA and swap, to advanced ones like caps and swaption, not to mention those complicated OTC products. The complexity of this market has pricing a hard task.

Short rate models are among the first attempt and yet the most basic approach in pricing. By assuming a virtual 'instant rate' and a stochastic process that this rate follow, one can estimate a price for these contracts reasonably well. BDT, Hull-White and BK are successful examples of this model. However, in real practice, this type of model has it own deficiency embedded in its nature. By assuming a 'instant rate', this model only has one source of randomness and all forward rates are perfectly correlated. This implies that once parameters are set, rate structures could only make parallel shift. Obviously, this is not true. The term structure can change its slope as market condition change, and in extreme events, might be totally different from normal structures. Unless parameters are re-calibrated, one factor models are not predicting such changes. These suggest that we need more advanced models are can incorporate more randomness.

Libor market models can suit this demand well. By allowing all forward rates to be random, it brings ample source of randomness, thus allowing any shape of the term structure to occur. In the most basic form, LMM assumes that all forward rates are random, though they may be correlated. If too much degree of freedom is a concern, more advanced implementations can choose a reduced form representation, either choosing some key point forward rates and interpolate others or using principle component analysis to back out all rates. In addition, LMM can be very flexible, allowing users to choose the form of volatility and correlation functions, which control the joint distribution of all rates. Such flexibility gives LMM much more power in fitting to market data and modeling real world situations. Theoretically, LMM would yield more accurate results even in pricing fundamental assets like swaptions. So long as the value of a contract depend on multiple rates, LMM is having some advantages as one factor models are exaggerating the correlations between rates.

This project, based on what we have learned during the class, tries to implement our version of the libor market model. We implemented 3 volatility functions and 2 correlation functions, and 3 simulation methods. In addition, our design makes sure that further generalization is possible. Users can define new class at will to add their choice to volatility and correlation functions, and a new simulation method, so long as they inherit the corresponding abstract base class. Finally, we applied our model to a real world contract.

This report is organized as follows. Section 2 work through a quick derivation of the libor market model. Section 3 introduces our implementation of the model, including the details of numerical procedure and calibration. Utilizing Object-Oriented Scheme, we implemented multiple choices for volatility, correlation and simulation method. Finally, we are going to discusses the procedure to price the chosen contract.

## Quick Derivation of Libor Market Models

### Model Set Up

Assume that we have $m$ forward rates and under the risk neutral measure, each subject to a Geometrical Brownian Motion.$$dF_i = \mu_i F_i dt + \sigma F_i dW_i$$ The solution to this stochastic differential equation is$$ \ln F_i( T ) = \ln F_i(0 ) - \int_0^T \frac{\sigma_k(t)^2}{2} dt + \int_0^T\sigma_i(t) dW_i(t)$$ One thing to note is that$dW_i$ might be correlated, that is$$dW_i(t) dW_j(t) = \rho_{i,j} dt$$ By properly choosing $\sigma_i$ and $\rho_{i,j}$, $F_i$ is well defined, with only one special attention that $u_i$ might not always be 0. Once numeraire asset is chosen, some forward rates might have a non-zero drift. The next subsection derives that drift terms for general forward rates.

### Drift Terms

Assume that we choose zero coupon bond maturing at$T_i$ as the numeraire asset, then the$i^{th}$ forward rate is a martingale. Further assume that $T_k \gt T_i$. Since forward rate agreement $F_kP(t,T_k)$ and zero coupon bond$P( t, T_k )$ are tradable assets, it follows from financial theorem that the following 2 variables are martingales

$$
 Y_k 
 = \quad \frac{P(t,T_k) }{P( t, t_i )}
 = \quad \frac{1}{ \prod_{j=i+1}^k (1 + F_j\tau_j ) } \\
 Z_k
 = \quad \frac{F_kP(t,T_k) }{P( t, t_i )}
 = \quad \frac{F_k}{ \prod_{j=i+1}^k (1 + F_j\tau_j ) } \\
$$

Given that $dF_k = \mu_k F_k dt + \sigma F_k dW_k$ and $Y_k$ is martingale, by product rule it must follow that$$dY_k = 0 dt - Y_k \sum_{j=i+1}^k {\frac{\tau_j}{1 + F_j\tau_j}\sigma_j F_j dW_j }$$ Then $dZ_k = d(F_k Y_k) = F_kdY_k + Y_k dF_k + dF_kdY_k$ would equal to$$ dZ_k = ( Y_k \mu_k F_k - Y_k \sum_{j=i+1}^{k}{ \frac{\tau_j}{1+F_j\tau_j} F_j \rho_{j,k} F_k F_j \sigma_k \sigma_j} ) dt + \cdots$$ Since $Z_k$ is a martingale, the drift term must equal to 0, so it must hold that$$ \mu_k = \sum_{j=i+1}^{k}{ \frac{\tau_j}{1+F_j\tau_j} F_j \rho_{j,k} F_j \sigma_k \sigma_j}$$

For $T_k \lt T_i$, similar analsis can be performed and the result is$$ \mu_k = -\sum_{j=i+1}^{k}{ \frac{\tau_j}{1+F_j\tau_j} F_j \rho_{j,k} F_j \sigma_k \sigma_j}$$

## Implementing Libor Market Models

As discussed before, Libor Market Models can be very different in set-up. Choices of volatility parameters, correlation functions and simulation methods would have effect on simulating process, and on pricing results. Therefore, an object-oriented scheme is considered optimal to produce readable and flexible code. This section is going to introduce the structure of our code. Basically, we recognized 3 fundamental elements in setting up a LMM, that is, volatility function, correlation function and simulation method. By defining an interface for these elements, we believe that our code is quite robust while open to extensions.

### Volatility Functions
Volatility function is the first building block of a Libor Market Model. At any given time point, a volatility function can be invoked to calculate the instant volatility of forward rates. Notice that there may be multiple forward rates and thus multiple volatility. Therefore, we require that 2 version of `get` functions. One of them get volatility for all alive forward rates, while another version get volatility for a specific forward rates. On other hand,  a volatility function should be able to calibrate to market data. The interface is designed so that volatility function would produce results closest to target volatility. In practice, this target volatility is usually the implied volatility of caplets maturing at different dates. Thus, we would come up with the following interface and any volatility function must implement these 3 functions.
``` python
class volatility( metaclass = ABCMeta ): 
    @abstractmethod
    def get( self, curTime, criticalTimePoint ):
        ...
    @abstractmethod
    def getI( self, curTime, criticalTimePoint, i ):
        ...
    @abstractmethod
    def calibrate( self, tarVol, criticalTimePoint, tau ):
        ...
```
In this project, we implemented 3 volatility function: function 2, 6 and 7 that are introduced in the lecture notes. All the 3 functions represents some sort of 'time-homogeneity' but this is not required quality for future volatility function classes.

`vol2`  is the volatility function class representing function 2. It has a volatility list inside itself. When `get` function is called and there are k alive forward rates, it would return the first k elements in the list. When `getI` is called, it would directly return the$i^{th}$ volatility without checking if the corresponding rate is still alive. The calibration is straight forward as introduced in the lecture notes:

$$
 \sigma_{Balck,1}^2 T_1
 = \quad \eta_1^2 \tau_1 \\
 \sigma_{Balck,2}^2 T_2
 = \quad \eta_2^2 \tau_1 + \eta_1^2 \tau_2\\
 \sigma_{Balck,3}^2 T_3
 = \quad \eta_3^2 \tau_1 + \eta_2^2 \tau_2 + \eta_1^2 \tau_3\\
$$

`vol6` is the volatility function class representing function 6. It has 4 parameters$a, b, c, d$ and instant volatility is calculated as$$ \sigma_{ins} = [ a( T_{i-1} - t ) + d ] e^{-b(T_{i-1} - t ) } + c$$ `get` and `getI` function would return the instant volatility. The calibration function minimizes the square difference of BS implied volatility and the integral of instant volatility functions$$ \min_{a,b,c,d} \sum_i \sigma_{Black,i}^2 - \frac{1}{T_i} \int_0^{T_i} \{ [ a( T_{i-1} - t ) + d ] e^{-b(T_{i-1} - t ) } + c \}^2dt$$

It is also restricted that $a > 0$ and $b>0$.

`vol7` is the volatility function class representing function 7. Similar to `vol6`, it has 4 parameters. In addition, it has a list of $\phi$ to ensure calibrating to market data. When `get` and `getI` function are called, the instant volatility is returned$$\phi_i \{ [ a( T_{i-1} - t ) + d ] e^{-b(T_{i-1} - t ) } + c \}$$ We do recognize that there are several methods to calibrate this volatility function, since there are $n+4$ parameters to calibrate to$n$ market data. The implementation here is calibrating the $4$ parameters first like what we have done in `vol6`. Then use $\phi$ to make an exact fit.

$$
 \min_{a,b,c,d} 
 \sum_i \sigma_{Black,i}^2 - \frac{1}{T_i} \int_0^{T_i} \{[ a( T_{i-1} - t ) + d ] e^{-b(T_{i-1} - t ) } + c \}^2 dt \\
 \phi_i^2
 = \quad \frac{\sigma_{Black,i}^2 T_i} {\int_0^{T_i} \{[ a( T_{i-1} - t ) + d ] e^{-b(T_{i-1} - t ) } + c \}^2 dt } \\
$$

### Correlation Functions
Together with volatility functions, correlation functions defined the characteristic of a Libor Market Model. Similar to volatility function, a correlation function should return a matrix upon request. The size of this square matrix is the same as number of alive forward rates. Also, there is an interface to calibrate to target correlation matrix. 
``` python
class correlator( metaclass = ABCMeta ):
    @abstractmethod
    def get( self, curTime, criticalTimePoint ):
        ... 
    @abstractmethod
    def calibrate( self, tarCorr, criticalTimePoint, tau ):
        ...
```
An important class of correlation function is parametric correlation function. That is, the correlation function generated according to pre-specified functions and parameters. A convenient interface for these functions is provided. The `get` and `calibrate` functions are implemented at this stage. However, all derived classes must define several relevant functions. 
``` python
class parametricCorrelator( correlator ):
    def get( self, curTime, criticalTimePoint ):
        # implemented
    def calibrate( self, tarCorr, criticalTimePoint, tau ):
        # implemented
    def __errToTarCorr( self, x, tarCorr, ctp ):
        ...
    @abstractmethod
    def getPara( self ):
        ...
    @abstractmethod
    def setPara( self, x ):
        ...
    @abstractmethod
    def getBound( self ):
        ...
    @abstractmethod
    def corrFun( self, Ti, Tj ):
        ...
``` 
However, there is an implicit restrictions on correlation functions, time-invariantness. It is assumed that the correlation function between any 2 forward rates is constant over time. While this choice sounds very restrictive, there are some reasons for it.

1.  Making correlations time variant would make calibrating to market data very complicated. For example, when fitting to swaption data, an integral in the form $\int \rho_{i,j} \sigma_i(t) \sigma_j(t) dt$ will be evaluated. If $\rho_{i,j}$ is time dependent, then this integral could be quite hard to calculate.
2.  Introducing time variance would introduce some uncontrollable risk. First, some numbers might be greater than 1. Second, there might be over-fitting. Finally, the matrix might not be positive definite.

In addition, although there is an interface to calibrate to target correlation matrix, we are not using it in this project. Originally, it is planned that first calibrating to volatility function with caplet implied volatility and the decide an optimal correlation matrix from swaption data. Then invoke the `calibrate` function in correlation function class to fit to this matrix. However, this is quite complicated in implementation and 2 step optimization would be both inefficient and inaccurate. Therefore, calibrating this way is useless if not detrimental. In this project, another calibration procedure is adopted. First, calibrate volatility function using caplet implied volatility. Second, optimize the parameters of a `parametricCorrelator` so that inferred swaption implied volatility matches market data as closely as possible. In this way, only one optimization is invoked and overall accuracy is ensured.

The first 2 parametric correlation functions in the lecture note are implemented here, that is
$$
 \rho_{i,j} 
 = \quad e^{ -\beta | T_i - T_j | }
 \beta \geq 0 \\
 \rho_{i,j} 
 = \quad \rho_{\infty} + ( 1 - \rho_{\infty} )e^{ -\beta | T_i - T_j | }
 \beta \geq 0 \\
$$

### Simulation Method

Unlike volatility and correlation functions, a simulation method has a base class named `LiborMarketModel`. It might be surprising that simulation methods have such a name. However, we believe that this design fits our need quit well. In our design, properties of forward rates are determined by volatility and correlation function, but not simulation methods. The later one only affect how we draw a path for all rates. Therefore, it is designed that a `LiborMarketModel` would have 2 members, `volatility` and `correlator`, and one member function to do simulation, `simulate`. To implement a new sampling strategy, just derive a class from `LiborMarketModel` and then over-write this function.

This class can calculate drifts of forward rates. When a model's initial state, volatility function and correlation functions are determined, the drift can be calculated given a chosen numeraire index. Therefore, `LiborMarketModel` has an implemented function named `getDrift`.   

On the other hand, a model should be calibrated to market data before put into use. Generally speaking, calibration method depends on the choice of correlation functions and thus a calibration function should be virtual. However, in this small project, we only implemented 2 correlation function which can be calibrated in the same way, so the function is implemented here. That is not saying that our implementation rejects extension. Indeed, this method can be accommodate to any parametric correlation functions.

A little bit more word about calibration. This function takes 2 parameters, `capletBSVol` and `swaptionBSVol`. The former one is the Black-Scholes implied volatility for caplets and the length of this list should be the same as number of forward rates. The second one, however, is not that straight forward. Market data from Bloomberg is usually organized as a table, with rows are for each maturity and columns for the tenor. However, we need to reshape this matrix into a list before putting into calibration function. Every element in this list is organized as $(T_0, T_n, impv)$. Given $T_0$ and $T_n$, we can calculator inferred variance by utilizing Rebonato's approximation using the formula$$ \sigma_{infer}^2 = \sum_{i,j=1}^{n} \frac{ \omega_i \omega_j f_i(0) f_j(0)}{S_{T_0,T_n}^2(0)} \int_0^{T_n}\rho_{i,j}\sigma_i(t)\sigma_j(t)dt$$ The idea of calibration is to minimize the square error of difference between implied volatility and inferred swaption volatility.$$\min \sum_i ( \sigma_{black,i}^2 T_i - \sigma_{infer}^2) ^2$$

Taking all this into consideration, we came up with an interface looks like: 
``` python
class LiborMarketModel( metaclass = ABCMeta ):
    def __init__( self, iniState, volCalc, corrCalc, criticalTimePoint, tau, numeraireIndex ):
        ...
    def getDrift( self, rateStruct ):
        # implemented
    def calibrate( self, capletBSVol, swaptionBSVol ):
	    # implemented
	@abstractmethod
    def simulate( self, finishTime ):
	    ...
```
Initiating `LiborMarketModel` is not possible as it is an abstract base class, but one may need initialize it when creating a new derived class. When this is the case, one need to provide the following information:
| Name | Meanning |
| :---       | :---                           |
| iniState   | initial state of forward rates |
| volCalc    | volatility function            |
| corrCalc   | correlation function           |
| criticalTimePoint | reset dates of different forward rates, expressed in floating numbers |
| tau        | difference of time between reset dates |
| numeraireIndex | index of numeraire forward rate, is an integer |

In this project, 3 type of simulator is implemented, short jumper, predictor corrector and iterative predictor corrector.

The first one is short jumper. It utilizes Euler Scheme in simulating the stochastic differential equation. Each time, it move forward a small step and generate a vector of new values. Because the drift terms in Libor Market Models are path dependent, Euler Scheme would be inaccurate if time step is large. Therefore, it is contracted that when using Short Jumper, time-step is between $0$ and $\frac{1}{12}$. If time step is greater maximum time step, it would be automatically reset to $\frac{1}{12}$. 
``` python
class ShortJumper( LiborMarketModel ):
    maxTimeStep = 1.0 / 12
```
The Euler Scheme simulation method is quite straight forward. While it has not reached the finish time, calculate the current drift, volatility and correlation matrix. Then generate a vector of random numbers $X \sim N( (\mu - \frac{1}{2}\sigma^2)dt, \Sigma dt )$. Then calculate new forward rates using $$f_{t+dt} \quad = \quad f_te^{X}$$

The second one is predictor-corrector. During each simulation, a vector of random variables is first generated and new forward rates are calculated like short jumper. However, instead of moving forward, calculate a new drift based on new rate value. Then final result is generated setting drift to be the average of old drift and new drift and the same random vector.
$$
 \mu_{old}  
 = \quad \mu( f_t, t )  \\
 \hat{ f_{t+dt} }
 = \quad f_{t} e^{\mu_{old} - \frac{1}{2}\sigma_{t}^2 dt + X } \\
 \mu_{new}
 = \quad \mu( \hat{ f_{t+dt} }, t ) \\
 \bar{ \mu }
 = \quad \frac{1}{2} ( \mu_{old} + \mu_{new } ) \\
 f_{ t + dt } 
 = \quad f_{t} e^{\bar{\mu} - \frac{1}{2}\sigma_{t}^2 dt + X } \\
$$

The third type of simulator is iterative predictor-corrector. It has a similar set up and simulating method as predictor-corrector. However, instead of reproducing drift for every forward rate simultaneously, it does the job in a sequential manner. Starting from numeraire forward rate which has a zero drift, it calculates the new forward rate. Then it moves  on to calculate drift of forward rates whose drift depend solely on numeraire forward rates, taking the average of drift based on current forward rate and new forward rates. That is, a forward rate is only updated when all new value of its dependency is known.

First, at time $t$, generate drift for all rates$$ \mu_{old} = \mu( f_t, t )$$
Second, let $i^{th}$ forward rate be the numeraire, the drift is always $0$, and future value is$$ f_{t+dt,i} = f_{t,i} e^{ -\frac{1}{2} \sigma_{t,i}^2 + \sigma_{t,i} Z }$$
Third, for $j = i-1, i-2 \cdots 1$ and then $j = i + 1, i+2, \cdots m$, sequentially, 
$$
 \mu_{new, j} 
= \quad \mu( f_{t,i}, f_{t,i-1}, \cdots, f_{t, j + 1}, t ) \\
 \bar{ \mu_{j } }
 = \quad \frac{1}{2} ( \mu_{old, j} + \mu_{new,j } ) \\
 f_{ t + dt, j } 
 = \quad f_{t,j} e^{\bar{\mu_{j}} - \frac{1}{2}\sigma_{t}^2 dt + Z } \\
$$
