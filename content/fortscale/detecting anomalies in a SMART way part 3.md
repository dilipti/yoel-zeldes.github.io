Title: Detecting Anomalies in a SMART Way (Part Three)
Slug: detecting-anomalies-in-a-SMART-Way-part-three
Date: 2016-10-20 21:00
Tags: statistics, data-science, anomalies
Summary: Third part of a posts series about finding anomalous users.
cover: images/detecting-anomalies-in-a-SMART-Way/cover.png

*This post was originally published by me at the [Fortscale blog](https://blog.fortscale.com/detecting-anomalies-in-a-smart-way-part-3).*  
*Fortscale's product helps organizations eliminate insider threats by detecting anomalous user behavior.*

---

In the [first post](detecting-anomalies-in-a-SMART-Way.html) of this series I described how we at Fortscale use a personalized adaptive threshold
for triggering alerts. Each user’s activity is assigned a risk score (known as the SMART value) that triggers a SMART alert when crosses the user’s
threshold. We explained how the more anomalous activities a user performs through time, the higher his threshold gets.


In the [second post](detecting-anomalies-in-a-SMART-Way-part-two.html) I dived into technical details, explaining how we use the Bayesian inference
framework in order to calculate a score for a SMART value. We developed a formula for the probability of seeing a SMART value of at least $v$ for a
user with past SMART values $v_1, ... , v_n$:
$$\left(\dfrac{\beta_{prior} + \sum_{i=1}^{n} v_i}{\beta_{prior} + \sum_{i=1}^{n} v_i + v}\right)^{\alpha_{prior} + n}$$
Where $\alpha_{prior}$ and $\beta_{prior}$ are hyper-parameters of the model. In this post we will uncover how we calculate these values.

---

The prior is a very powerful attribute of the Bayesian inference framework; it allows one to incorporate her prior knowledge when calculating the probability.
This attribute is one of the reasons we like Bayesian inference so much. Recall from the first post of the series that we had some challenges with users who
never acted anomalously, and hence their past SMART values are all zeros. Such users will surely trigger an alert for any positive SMART value. We can make
use of the prior to prevent this from happening.

First, let’s understand how the prior and posterior work together. The prior is the probability function over $\lambda$ given no data. Once we observe some
data (user generated SMART values), we update our prior and get the posterior, which is again a probability function over $\lambda$. Since we chose a
[conjugate prior](https://en.wikipedia.org/wiki/Conjugate_prior) (which is the Gamma distribution in the case of the exponential model), we gain the desired
property that the update is simply a matter of updating the Gamma distribution’s parameters. If more data is collected, we can update the posterior’s
parameters once again. It’s an ongoing process of updating the Gamma distribution’s parameters.

The Gamma distribution’s parameters are $\alpha$ and $\beta$ (or $\alpha_{prior}$ and $\beta_{prior}$, since we use the Gamma as a prior). If you’re
not familiar with this parameterization, you’re encouraged to read about it
[here](https://en.wikipedia.org/wiki/Gamma_distribution#Characterization_using_shape_.CE.B1_and_rate_.CE.B2).
$\alpha_{prior}$ can be interpreted as being the number of SMART values virtually observed by a user, while $\beta_{prior}$ is their sum.
The user’s real observed data is essentially merged with the virtual data represented by the prior. So if we take the previous example of a user
with only zeros as his data, and give a prior with high $\beta_{prior}$ to $\alpha_{prior}$ ratio, we get the same effect as if the user had some
high values, which results in a higher threshold. This is exactly what we wanted to accomplish.

---

Following the above, we should choose high $\beta_{prior}$ to $\alpha_{prior}$ ratio. But what should this ratio be? Our goal is to do what an analyst
would do when inspecting user activity. The analyst is familiar with the organization and knows which activities could be more interesting, since he has
the prior knowledge of how many anomalous activities are going on in the organization. If there’s no anomalous activity at all, any suspicious user
activity should be investigated. If anomalous activities is a matter of routine in the organization, the analyst’s interest threshold is higher.

We should simulate this effect using the prior. If we choose $\alpha_{prior}$ to be the number of SMART values in the organization and $\beta_{prior}$
to be their sum we’ll get the desired effect. Using this method, the prior represents the knowledge of the amount of anomalous activities taking place
in the organization.

---

There is one downside to this approach, namely, it causes $\alpha_{prior}$ to be pretty big, which leads to a prior that is too influential. In turn,
this will cause that the user’s data will only slightly affect the result probability, i.e. using the posterior will get almost the same results as
using the prior. This is not what we aimed for. We wanted the threshold to be personal, which means that the user’s data should affect his threshold.

The reason the prior is too influential is because it simulates SMART values for the user. If there are 10,000 SMART values in the organization
$\alpha_{prior}$ will be 10,000. If the user had only 100 SMART values, they will be outweighed by the simulated 10,000 SMART values.

If we want to decrease the prior influence, we should decrease $\alpha_{prior}$ while preserving the $\beta_{prior}$ to $\alpha_{prior}$ ratio.
The heuristic we chose after experimenting with real life data is to use some reasonable small number, e.g. 20, and to update $\beta_{prior}$ to
be $\alpha_{prior}$ times the average of the organization’s SMART values.

The math behind our choice of the number 20 is simple. The mean of the prior Gamma distribution is $\alpha_{prior}$ divided by $\beta_{prior}$.
The choice of 20 doesn’t affect the mean since we scale $\beta_{prior}$ accordingly. The variance however, is $\alpha_{prior}$ divided by
$\beta_{prior}$ squared. Reducing the prior’s “influence” to 20 causes the variance to increase. The intuition behind the prior’s variance
is that small variance is analogous to an old stubborn prior that thinks it knows best; when it tells you the expected value is $\alpha_{prior}$
divided by $\beta_{prior}$, it means exactly that. Big variance on the other hand is like an educated young prior. Such prior knows what the
expected value is, but it is not that stubborn and can be “convinced”. Such prior knows there are surprises down the road and consequently
allows some uncertainty. 

The above is what I meant by the prior’s “influence”: more influence means you should listen to it, and less influence means it gives you some hint,
but you should give more weight to the actual user’s data. Choosing the right prior allows us to enjoy both worlds. We get a reasonable prior which
takes into account the amount of anomalous activities going on in the organization, while not compromising the desired influence of the user’s data.