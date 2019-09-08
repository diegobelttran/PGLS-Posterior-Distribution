# PGLS-Posterior-Distribution
Comparing phenotypic traits with speciation/extinction rates using samples from a posterior distribution, with Phylogenetic Generalized Least Squares in R.

In my case I used color evolution rates obtained from BAMM as phenotypic traits and speciation rates obtained also from BAMM. BAMM is a Bayesian approach, hence, after running analyses you obtain a a posterior distributon of different trait evolution configurations.

What I aim to do here is to run a PGLS between color evolution rates and speciation rates. These rates will be averaged by clade. These clades will be chosen according to each BAMM configuration from a sample from the posterior distribution. So for example, if in one configuration form the posterior only one significant shift change was identified by BAMM, there will only be two clades with different rates. All tips within each clade will share their evolution rates. After averaging the rates of all tips within each clade I get two points in my color evolution rates vs. speciation rates plot, with these two points I run a PGLS and save the coefficients I get and repeat with the next sample from the posterior.

After repeating the same process with a good sample from the posterior, say 100, I get a sample of 100 PGLS coefficients and I can plot a histogram to see the general distributions of relationships between trait evolution and speciation, taking into account phylogenetic uncertainty.

This code is very rough and long I know, any help shortening and improving it is appreciated.

THIS SCRIPT IS STILL BEING TRANSLATED AND COMMENTED
