# Fundamentals

## Essence

In one word, **machine learning is looking for functions**.

### Types of functions

* Regression: the function output is a scalar
* Classification: given options, the function output is the right choice
* Structured Learning: create something with structure(image, document)

### Process

#### Function with Unknown Parameters

​	***y = b+wx*** (b is bias, w is weight, x is feature)

#### Define Loss from Training Data

​	**Loss** is a function of parameters: ***L(b,w)***， loss indicates how good a set of value is

​	**label** is the training data value corresponding to feature

![Loss](C:\Users\xh254\Desktop\fafacao_blog\MachineLearning_HongYiLee\media\Loss.png)



#### Optimization

Find a a set of parameters to get the minimum Loss

Method: **Gradient Descent**

![GradientDescent](C:\Users\xh254\Desktop\fafacao_blog\MachineLearning_HongYiLee\media\GradientDescent.png)

**η: Learning Rate (Increament in each round of update)**



## Sophisticated Models

linear models have severe limitation, they are too simple

![piecewise](C:\Users\xh254\Desktop\fafacao_blog\MachineLearning_HongYiLee\media\piecewise.png)