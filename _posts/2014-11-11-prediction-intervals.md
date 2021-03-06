---
layout: post
title: Confidence intervals vs. prediction intervals in R
subtitle: What do they mean?
published: false
---

In R speak, there are confidence intervals and prediction intervals. Using loose terminology, confidence intervals refer to uncertainty in the coefficients and prediction intervals refer to the expected variation in additional observations---prediction intervals include both uncertainty in the coefficients and the residual sampling noise.

If you recorded the heights of a group of students in a class and calculated the mean, you might want to know how certain you are that your sample mean (from your class) matches what you'd get if you went out and recorded the height of all students at the same age across many schools (the population). The "confidence" interval helps you with this. As you collect more observations, your certainty in the mean increases and the confidence interval decreases.

If, instead, you wanted to know the expected range of student heights you'd observe if you surveyed a second class room, you'd want to use a prediction interval. Even if you had sampled 10,000 students, the expected variability in observations you'd get in another class room would be about the same. Prediction intervals can be estimated more precisely as you gain more data, but they shouldn't systematically get smaller.

Let's use a small example in R to illustrate this. First, we'll simulate some data and fit a linear model with only an intercept. This is equivalent to estimating the population mean.


```r
x <- rnorm(50, mean = 1.5, sd = 0.6)
m <- lm(x ~ 1)
```

We can get the two kinds of intervals in R like this:


```r
conf <- predict(m, se.fit = TRUE, interval = "confidence")
pred <- predict(m, se.fit = TRUE, interval = "prediction")
```

If you want to see more details on these options, see `?predict.lm`.

Now we'll plot these intervals:


```r
plot(x)
abline(h = 1.5, col = "red")
abline(h = c(conf$fit[1, "lwr"], conf$fit[1, "upr"]))
abline(h = c(pred$fit[1, "lwr"], pred$fit[1, "upr"]), lty = 2)
```

![plot of chunk unnamed-chunk-3](/knitr-figs/unnamed-chunk-3-1.png) 

Notice how much wider the prediction intervals (dashed lines) are. They encompass most of the observed data points.

Now, let's do that repeatedly to see more clearly what these intervals refer to. First, we'll write a little function to do what we did above. We'll simulate some data, fit a model, and get confidence intervals from that model. Later, we'll plot these.


```r
sim_intervals <- function(n = 20, mean = 2, sd = 1) {
  x <- rnorm(n = n, mean = mean, sd = sd)
  m <- lm(x ~ 1)
  conf <- as.data.frame(t(predict(m, se.fit = TRUE, interval = "confidence")$fit[1, ]))
  pred <- as.data.frame(t(predict(m, se.fit = TRUE, interval = "prediction")$fit[1, ]))
  names(conf) <- paste0("conf_", names(conf))
  names(pred) <- paste0("pred_", names(pred))
  data.frame(sample = seq_along(x), x, mean, sd, conf, pred)
}
```

And we'll write a ggplot2 function to plot the data:


```r
plot_intervals <- function(dat) {
  library("ggplot2")
  ggplot(dat, aes(sample, x)) + geom_point(cex = 1.1) + facet_wrap(~id) +
  geom_hline(aes(yintercept = mean), colour = "red", lwd = 1) +
  geom_hline(aes(yintercept = c(conf_lwr, conf_upr))) +
  geom_hline(aes(yintercept = c(pred_lwr, pred_upr)), colour = "grey80") +
  theme_bw() +
  theme(panel.grid.major = element_blank(),
    panel.grid.minor = element_blank(),
    panel.background = element_blank(),
    strip.background = element_blank())
}
```

We'll use the plyr package to run our function 20 times:


```r
set.seed(1)
out_n20 <- plyr::ldply(setNames(seq_len(20), seq_len(20)), function(i) sim_intervals(n = 20, mean = 2, sd = 0.5), .id = "id")
```

And again with more data:


```r
set.seed(1)
out_n100 <- plyr::ldply(setNames(seq_len(20), seq_len(20)), function(i) sim_intervals(n = 100, mean = 2, sd = 0.5), .id = "id")
```

And plot the first example:


```r
plot_intervals(out_n20)
```

![plot of chunk unnamed-chunk-8](/knitr-figs/unnamed-chunk-8-1.png) 

So, roughly 1 time out of 20 the confidence interval (black lines) should miss containing the true mean value (red line). We see that in panel 9.

And, approximately 95% of the sample dots should be inside the prediction interval (grey lines). In other words, on average 1 dot should be outside the grey lines in each panel. In general that seems about right too. 

Using a prediction interval on existing data doesn't make a lot of sense. You already know the observed data after all. Typically, you'd use the prediction interval for 'future' or 'additional' observations, especially in a scenario where you're doing more than just estimating a mean. Maybe you want to predict the variability in your observations at a different level of a predictor variable. Here, I'm showing them on the existing data just to illustrate a point. Further, this can be a useful way of checking your calculations to make sure they match the original data.

Let's compare the second example we ran above. In the second example, we had 100 observations instead of 20:


```r
plot_intervals(out_n100)
```

![plot of chunk unnamed-chunk-9](/knitr-figs/unnamed-chunk-9-1.png) 

You'll notice that the confidence intervals have gotten much tighter (although on average 1 out of 20 will still miss the mark). The prediction intervals are just as wide as before and on average encompass 95% of the observations.

