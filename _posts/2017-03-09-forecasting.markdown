---
layout: post
title:  "Forecasting Time Series Data"
date:   2017-03-09 20:44:03 -0500
categories: R ACF PACF holt winters train testing seasonal 
summary: Comparing forecasting methods for time series data with model selection in R. 
---

Let's look at the time series dataset in R called [co2](https://stat.ethz.ch/R-manual/R-devel/library/datasets/html/co2.html). This dataset measures monthly Mauna Loa atmospheric CO2 concentration from 1959 to 1997 totaling 468 observations. 

We will be forecasting the data using three methods then comparing the methods of forecast. How do you compare methods or their accuracy? We will slip the data up into two parts. The first 1 thorugh 444 oberservations will be called the training data. This is the portion of the data we will base our forecasting on. The last 5% of the data (ie. the last 24 observations) is called the test data. The training data will be used to predict the last 24 observations, then predicted values and test data can be compared to measure accuracy of the model. For what we want to do we need additional packages. If you have never before worked with them, remember to intall the packages before you can load them into your library. 
{% highlight r %}
library(TSA)
library(fArma)
library(forecast)
train <- co2[1:444]
test <- co2[445:468]
{% endhighlight %}

Let's take a look at the data. 
{% highlight r %}
plot.ts(co2)
par(mfrow=c(1,2))
acf(co2)
pacf(co2)
{% endhighlight %}
![data plot](/assets/Forecast/image1.png)
The time series data has a positive linear trend. There are no issues with heteroscedasticity. The constant up and down pattern indicates a seasonal trend of period 12. 

![acf pacf](/assets/Forecast/image2.png)
This is a unique ACF PACF combination that indicates nonstationarity. The ACF decays to zero at a very slow rate while the PACF is only really significant at lag 1. We expected to see signs of nonstationarity because of the non constant mean we previously noted. 

### Forecast 1 


The first method of forecasting we will try is called subset selection method. This is a computer method where we input the maximum AR and MA orders we believe may explain the model, here we will choose 15 for both. The computer program then runs through all possible ARMA model cominiations of those coefficients and returns the most probable models based on thier BIC values.
{% highlight r %}
sub1 <- armasubsets(diff(diff(train,12)),nar=15,nma=15)
plot(sub1)
{% endhighlight %}
![acf pacf](/assets/Forecast/image4.png)
Based on our output, we believe the best model is an ARIMA(9,1,12). The 1 indicates that we applied a differencing to the data. The AR component is 9 because that is the highest order cofficient however there is also a nonzero coefficient value for phi 1. Similarly, the MA component is 12 because that is the highest order cofficient however there is also a nonzero coefficient value for theta 2. The Arima function below gives us the values for the coefficients of the ARIMA model. 
{% highlight r %}
arma1 <- Arima(train,order=c(9,1,12), seasonal=c(0,1,0),fixed=c(NA,rep(0,7),NA,0,NA,rep(0,9),NA))
arma1
{% endhighlight %}
![acf pacf](/assets/Forecast/image5.png)

WE can see below that the forecast isnt doing a very good job at following the pattern of the data in the prediction. Additinally the confidence intervals are verly large. While all of the test data points are in the respective confidence intervals, this model does not look ideal. 
acf(diff(diff(train,12)),lag.max=40,main="Differenced co2")
pacf(diff(diff(train,12)),lag.max=40,main="Differenced co2")

![ci graph](/assets/Forecast/image6.png)
![ci](/assets/Forecast/image7.png)


### Forecast 2 


The next method of forecasting will be to identify potential SARIMA models from the ACF and PACF. Then fit the candidate models nad compare their AICc values to choose a final SARIMA model to run the forecast on. 

We need to look at the ACF/PACF of the differenced data. 
{% highlight r %}
ddtrain <- diff(diff(train),12)
par(mfrow=c(1,2))
acf(ddtrain,lag.max=40,main="Differenced co2")
pacf(ddtrain,lag.max=40,main="Differenced co2")
{% endhighlight r %}
![acf pacf](/assets/Forecast/image8.png)
Both the ACF and PACF decay to zero for the first few lags which indicates that possible models could be AR(1) or MA(1) or ARMA(1,1). As for the seasonal component, the ACF cuts off after lag 1 (aka lag 12 but this is the first seasonal lag) and the PACF decays to zero at the seasonal lags (every twelfth lag). These suggest an MA(1) model is appropriate for the seasonal component. 
{% highlight r %}
Arima(ddtrain, order=c(0,1,1), seasonal=c(0,1,1))
Arima(ddtrain, order=c(1,1,0), seasonal=c(0,1,1))
Arima(ddtrain, order=c(1,1,1), seasonal=c(0,1,1))
{% endhighlight r %}
![acf pacf](/assets/Forecast/image9.png)
Of the three models we tested, SARIMA(1,1,1)x(0,1,1)[12] performed best with the lowest AICc value of 374.47. Below we see the model is not very good at predicting the testing data. The model does not keep the same seasonal pattern and is not even behaving in a positive linear manner. The confidence intervals are also very large. 
![acf pacf](/assets/Forecast/image10.png)
![acf pacf](/assets/Forecast/image11.png)
 
### Forecast 3


 The third method of forecasting is called a Holt-Winters seasonal forecast. We specify an additive seasonal trend instead of multiplicative because there is no heteroscedasticity. The "gamma = T" condition specifies that there is a seasonal component to address. The Holt-Winters is a form of double exponential smoothing that is good to use when there is a linear trend and/or a known seasonal component.  

{% highlight r %}
hw <- HoltWinters(ts(train,frequency=12),gamma = T, seasonal = "additive")
forecast(hw, h=24)
plot(forecast(x, h=24))
{% endhighlight r %}
The Holt-Winters is a good predictor. The trend and seaonality patters are consistant with the previous data and the confidence intervals are narrow. Additionally, all 24 testing data points lie within the 95% prediction interval. 
![acf pacf](/assets/Forecast/image12.png)
![acf pacf](/assets/Forecast/image13.png)

### Comparing Forecastings 
To compare the forecast we will use two common criteria that quantify how much error the model produced. The lower the RMSE and MAPE the better the model was at predicting the testing data.

{% highlight r %}
HWfcast1 <- forecast(arma1,h=24)
HWerr1 <- test-HWfcast1$mean
HWrmse1 <- sqrt(mean(HWerr1^2))
HWmape1 <- mean(abs((HWerr1*100)/test))
HWrmse1
HWmape1
{% endhighlight r %}

{% highlight r %}
ff2 <- Arima(train, order=c(1,1,1), seasonal=c(0,1,1))
HWfcast2 <- forecast(ff2,h=24)
HWerr2 <- test-HWfcast2$mean
HWrmse2 <- sqrt(mean(HWerr2^2))
HWmape2 <- mean(abs((HWerr2*100)/test))
HWrmse2
HWmape2
{% endhighlight r %}

{% highlight r %}
ff3 <- HoltWinters(ts(train,frequency=12),gamma = TRUE,seasonal="additive")
HWfcast3 <- forecast(ff3,h=24)
HWerr3 <- test-HWfcast3$mean
HWrmse3 <- sqrt(mean(HWerr3^2))
HWmape3 <- mean(abs((HWerr3*100)/test))
HWrmse3
HWmape3
{% endhighlight r %}

 ![acf pacf](/assets/Forecast/image14.png)
 As we expected the Holt- Winters forecasting was the best. Next, the ARIMA(9,1,12) model prformed slightly better than the SARIMA(1,1,1)x(0,1,1)[12] model. A second look a the forecasting graphs for these two models confirms their performances. The SARIMA model had no trend or seasonal pattern while the ARIMA model did have a slight seasonal pattern in its prediction. 


