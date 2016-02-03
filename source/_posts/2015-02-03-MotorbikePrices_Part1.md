---
title: "Motorbike Prices"
author: "atwam"
date: "3 February 2016"
output: html_document
---
The basis of my data is a request on autotrader for motorbikes which cost more than £500 (to avoid all the "wanted") posts, and with more than 200cc.



Let's have a look at our data : 

{% highlight r %}
kable(summary(data))
{% endhighlight %}



|   |   title         |   brand         |    price     |   distance   |        seller_type  |category |     year    |   mileage     |      cc       |    url          |description      | panniers     |   abs        |   search        |     age       |
|:--|:----------------|:----------------|:-------------|:-------------|:--------------------|:--------|:------------|:--------------|:--------------|:----------------|:----------------|:-------------|:-------------|:----------------|:--------------|
|   |Length:17043     |Length:17043     |Min.   :  500 |Min.   :  1.0 |Private seller:  558 |:16950   |Min.   :1926 |Min.   :     1 |Min.   : 191.0 |Length:17043     |Length:17043     |Mode :logical |Mode :logical |Length:17043     |Min.   : 0.000 |
|   |Class :character |Class :character |1st Qu.: 4137 |1st Qu.: 54.0 |Trade seller  :16485 |C:   57  |1st Qu.:2008 |1st Qu.:  2000 |1st Qu.: 650.0 |Class :character |Class :character |FALSE:16123   |FALSE:13347   |Class :character |1st Qu.: 1.000 |
|   |Mode  :character |Mode  :character |Median : 6000 |Median :114.0 |NA                   |D:   36  |Median :2012 |Median :  7266 |Median : 900.0 |Mode  :character |Mode  :character |TRUE :920     |TRUE :3696    |Mode  :character |Median : 4.000 |
|   |NA               |NA               |Mean   : 6857 |Mean   :116.7 |NA                   |NA       |Mean   :2010 |Mean   : 10844 |Mean   : 921.3 |NA               |NA               |NA's :0       |NA's :0       |NA               |Mean   : 5.573 |
|   |NA               |NA               |3rd Qu.: 8695 |3rd Qu.:168.0 |NA                   |NA       |3rd Qu.:2015 |3rd Qu.: 15900 |3rd Qu.:1170.0 |NA               |NA               |NA            |NA            |NA               |3rd Qu.: 8.000 |
|   |NA               |NA               |Max.   :55950 |Max.   :445.0 |NA                   |NA       |Max.   :2016 |Max.   :333902 |Max.   :3072.0 |NA               |NA               |NA            |NA            |NA               |Max.   :90.000 |
|   |NA               |NA               |NA            |NA            |NA                   |NA       |NA's   :521  |NA's   :1832   |NA's   :19     |NA               |NA               |NA            |NA            |NA               |NA's   :521    |

# Impact of the age on the price

{% highlight text %}
## Warning: Removed 521 rows containing non-finite values (stat_smooth).
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-2-1.png)

What can we see on this graph ?
- Prices go down almost linearly during the first 15 years.
- After that, you can expect prices to remain kind-of stable for bikes which are just about to become vintage.
- Motorbikes more than 30 years old start to see a constant rise in value.
- Cat C and Cat D bikes have a value that is constantly lower than the value of non-category motorbikes. They decrease in value steadily.

Now, let's focus on bikes which are less than 15 years old, not in a category.

{% highlight r %}
data = subset(data, age <= 15 & category == "")
{% endhighlight %}

We can evaluate this discount :

{% highlight r %}
pander(lm(price ~ age, data))
{% endhighlight %}


--------------------------------------------------------------
     &nbsp;        Estimate   Std. Error   t value   Pr(>|t|) 
----------------- ---------- ------------ --------- ----------
     **age**        -442.6       6.81      -64.99       0     

 **(Intercept)**     8961       40.13       223.3       0     
--------------------------------------------------------------

Table: Fitting linear model: price ~ age

We see whe should expect a loss of value of £442/year, but it makes more sense to assume this loss in value should be a percentage of the original value, which would be 0.0493249, or about 5%/year for the first 15 years.

# Impact of brands
As a first approximation, we can take the first word of the ad title to get the brand. Brands that use several words will be trimmed, but that should be good enough for now.

{% highlight r %}
data$brand = as.factor(word(data$title))
brands = data %>% group_by(brand) %>% tally()
big_brands = droplevels(brands[n > 200]$brand)
data$big_brand = data$brand %in% big_brands
ggplot(data[data$big_brand]) + geom_smooth(aes(y=price, x=age, color=brand), se = FALSE)
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-4-1.png)
Here we can see not only the difference in average price between brands (be careful, I haven't distinguished between models here), and how these decay with time. We can see that Harleys are consistently more expensive, but seen to hold their value relative to other bikes quite well. Ducatis see a sharp drop at around 5 years.
BMW are more expensive than the other brands as well in their first 7 years or so, but then drop in price to be actually cheaper than other brands. My guess is that because they are more expensive, they tend to be favoured by people able to buy them new, but the second-hand market for them isn't that great.

# Engine size

{% highlight r %}
ggplot(data) + geom_smooth(aes(y=price, x=cc))
{% endhighlight %}



{% highlight text %}
## Warning: Removed 9 rows containing non-finite values (stat_smooth).
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-5-1.png)
Not much to see here by default, we can just see that the price of a bike grows (almost linearly) with the engine size. That is until 1800cc (Goldwings), then beyond 2000cc you enter the realm of the Rocket III which is just less expensive than the goldwing.

# Mileage
Some bikes seem to have crazy mileage (300k), so let's stick to stuff with reasonable values, less than 100k.

{% highlight r %}
ggplot(subset(data, mileage < 100000)) + geom_smooth(aes(y=price, x=mileage))
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-6-1.png)
Big surprise, the price drops as the mileage increase. The effect seems almost linear to start with, then seems to decay. We can probably work with a linear decay for the first 25k, beyond that it just makes sense to use a log decay...

{% highlight r %}
summary(lm(price ~ mileage, subset(data, mileage < 25000)))
{% endhighlight %}



{% highlight text %}
## 
## Call:
## lm(formula = price ~ mileage, data = subset(data, mileage < 25000))
## 
## Residuals:
##    Min     1Q Median     3Q    Max 
##  -7708  -2237   -612   1667  47778 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  8.208e+03  4.409e+01  186.18   <2e-16 ***
## mileage     -1.569e-01  4.401e-03  -35.65   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3349 on 12764 degrees of freedom
## Multiple R-squared:  0.09055,	Adjusted R-squared:  0.09048 
## F-statistic:  1271 on 1 and 12764 DF,  p-value: < 2.2e-16
{% endhighlight %}



{% highlight r %}
summary(lm(price ~ log(1 + mileage), data))
{% endhighlight %}



{% highlight text %}
## 
## Call:
## lm(formula = price ~ log(1 + mileage), data = data)
## 
## Residuals:
##    Min     1Q Median     3Q    Max 
##  -8961  -2347   -603   1613  48229 
## 
## Coefficients:
##                  Estimate Std. Error t value Pr(>|t|)    
## (Intercept)       9715.54      91.98   105.6   <2e-16 ***
## log(1 + mileage)  -366.54      10.97   -33.4   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3370 on 14061 degrees of freedom
##   (1442 observations deleted due to missingness)
## Multiple R-squared:  0.0735,	Adjusted R-squared:  0.07344 
## F-statistic:  1115 on 1 and 14061 DF,  p-value: < 2.2e-16
{% endhighlight %}
What we can see here is that the price of the bike seems to drop by 0.1569 per mile, or £156/1000miles.
Translating again to a percentage of the intercept, we get 0.0190058 or about 1.9% per 1000 miles.

# Mileage + age
Mileage and age are obviously very (read 0.6524094 = 65%) correlated, so we'll have to use both in our price regression.


{% highlight r %}
summary(lm(price ~ age * log(1+mileage), subset(data, mileage < 100000)))
{% endhighlight %}



{% highlight text %}
## 
## Call:
## lm(formula = price ~ age * log(1 + mileage), data = subset(data, 
##     mileage < 1e+05))
## 
## Residuals:
##    Min     1Q Median     3Q    Max 
##  -7869  -1921   -400   1292  47533 
## 
## Coefficients:
##                      Estimate Std. Error t value Pr(>|t|)    
## (Intercept)          8325.324     98.211  84.770  < 2e-16 ***
## age                    -3.961     45.923  -0.086    0.931    
## log(1 + mileage)       62.846     13.223   4.753 2.02e-06 ***
## age:log(1 + mileage)  -45.364      4.654  -9.748  < 2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3075 on 14055 degrees of freedom
## Multiple R-squared:  0.2289,	Adjusted R-squared:  0.2287 
## F-statistic:  1390 on 3 and 14055 DF,  p-value: < 2.2e-16
{% endhighlight %}
We can pretty much ignore the coef in front of age (low tstat), we get something that is roughly :
$ price = 8325 + lmileage * (62 - 45 * age) $

# Are private seller cheaper than trade sellers

{% highlight r %}
ggplot(data) + geom_smooth(aes(y=price, x=age, color=seller_type)) + scale_x_continuous(breaks = seq(0,90,10))
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-9-1.png)
The answer seems consistently *yes* !

# Other factors
## ABS

{% highlight r %}
ggplot(data) + geom_smooth(aes(y=price, x=age, color=abs)) + scale_x_continuous(breaks = seq(0,90,10))
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-10-1.png)
Well, we can see that ABS definitely seems to keep its increased price, at least for recent bikes. One has to be careful though, because ABS is usually found on higher end bikes, so this difference in price could just be a bias. One would have to do a price comparison for a specific model (with enough data).
## Distance to london

{% highlight r %}
ggplot(data) + geom_smooth(aes(y=price, x=distance))
{% endhighlight %}

![center](/figs/2015-02-03-MotorbikePrices_Part1/unnamed-chunk-11-1.png)
Hard to say anything here. Could be some bias from big cities that appear at some distance from London. Given we don't have the postcode for all ads, it's hard to go beyond this useless graph.

