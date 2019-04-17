---
layout: post
title: How to Explore and Construct a Data Narrative, Regression Analysis in R
---

Building a narrative around data is important for making a convincing argument about what the data shows. The best data narrative should be able to present all arguments succinctly and clearly. 

In this post, I walk through the construction of a data narrative, starting from the exploration of data and ending with a complete story told by the data. 

All the code I wrote to analyze and generate plots in R is shown alongside each section. The code is continued from the previous blog post on data cleaning and transformation.

### Hypothesis
Let's say we are an imaginary firm that is trying to decide if we are going to enter the home loans market. To understand if we should enter the market, we must explore the data to understand the general home loans trends occurring and the reasons behind them.

I **hypothesize that it is not currently a great time to enter the home loan market** (in 2014), since during this time period, the home loan market shrunk significantly.

### General Trends in Proposed Market

```r
#For reading in dt.loans, see my previous post on data parsing/cleaning/transformation

#Look at all loans, by year and loan purpose
dt.year.trend <- dt.loans[,list(loan.sum.usd.billions = sum(loan.usd)/1e9), 
                                          by = list(year,loan.purpose.desc)]

#Reshape this data so we can sum up the total
dt.year.trend.cast <- data.table(dcast(dt.year.trend, year ~ loan.purpose.desc, value.var = "loan.sum.usd.billions"))
dt.year.trend.cast <- dt.year.trend.cast[,Total := `Purchase` + `Refinance`]
dt.year.trend <- data.table::melt(dt.year.trend.cast, id.vars = "year", variable.name = "loan.purpose.desc", value.name = "loan.sum.usd.billions")

#Pick some numbers to use in the writeup
num.refinance.sum.2012 = dt.year.trend.cast[year == 2012, Refinance]
num.refinance.sum.2014 = dt.year.trend.cast[year == 2014, Refinance]
num.purchase.sum.2012 = dt.year.trend.cast[year == 2012, Purchase]
num.purchase.sum.2014 = dt.year.trend.cast[year == 2014, Purchase]
```

Between 2012 and 2014, total loans dropped significantly, going from 150 to 81 billion USD as our hypothesis assumed. However, that trend differed greatly by loan type. **Figure 2** shows that while refinancing loans changed by  -71%, house purchase loans actually increased by 19%.  This shows that **our hypothesis is not correct**; although the home loan market is shrinking, home purchasing loans are rising, and may represent a good opportunity. By 2014, more loans were being given for purchasing than for refinancing, which means home purchase loans is currently a more desirable product to pursue.

```r
#Line plot will illustrate the trend both generally and by loan purpose
ggplot(data=dt.year.trend, aes(x = year, y = loan.sum.usd.billions, color = loan.purpose.desc))+
  geom_line()+ geom_point(size = 2) +
  labs(x = "Year",
       y = "Total Loans (Billions)",
       title = "Figure 2. Total Loans by Year")+ theme(text = element_text(size=14), axis.text.x = element_text(angle=90)) + scale_color_discrete(name="Loan Type") + scale_x_continuous(breaks = c(2012:2014)) + scale_y_continuous(limits = c(0, 170))
```

<img src="/img/figure-2.png" width="700px"/>

### Market Size by State

Next, let's look at the trend by state. *Figure 3* shows the same trends apply across all states, with the total home loan market shrinking, but the purchase home loan market increasing. We also see that **the largest markets by far are VA and MD**. In fact, they dwarf the other markets, making up about 85% of the total market. This suggests that, as a new market entrant, we should **focus on just these two states** to maximize our access to the overall market while minimizing the regulatory costs associated with expanding to a different state.

```r
dt.state.trend <- dt.loans[,list(loan.sum.usd.billions = sum(loan.usd)/1e9), 
                                          by = list(year,loan.purpose.desc, state.short)]


#Reshape this data so we can sum up the total
dt.state.trend.cast <- data.table(dcast(dt.state.trend, year + state.short ~ loan.purpose.desc, value.var = "loan.sum.usd.billions"))
dt.state.trend.cast <- dt.state.trend.cast[,Total := `Purchase` + `Refinance`]
dt.state.trend <- data.table::melt(dt.state.trend.cast, id.vars = c("year", "state.short"), variable.name = "loan.purpose.desc", value.name = "loan.sum.usd.billions")

ggplot(data=dt.state.trend, aes(x = year, y = loan.sum.usd.billions, color = loan.purpose.desc))+
  geom_line()+ geom_point(size = 2) +
  labs(x = "Year",
       y = "Total Loans (Billions)",
       title = "Figure 3. Total Loans by State and Year")+ theme(text = element_text(size=14), axis.text.x = element_text(angle=90)) + scale_color_discrete(name="Loan Type") + scale_x_continuous(breaks = c(2012:2014)) + scale_y_continuous(limits = c(0, 90)) + facet_grid(~state.short)
```

<img src="/img/figure-3.png" width="700px"/>

### Establishing a General Trend

To understand this trend, we can begin by investigating if the trend can be attributed to a general factor, such as the economy or national interest rates, versus a local factor, like a change in regional real estate value or a home loan company changing their lending standards. I will proceed to show **the trend is in fact uncorrelated with any particular geographic location or loan company**. 

#### Metropolitan Area
While we've shown the trend is consistent across different states, we can look with even finer resolution at different metropolitan areas, characterized in this dataset as Metropolitan Statistical Areas/Metropolitan Divisions (MSA/MD). Looking at the metropolitan areas with the top 7 highest total home loan amounts, we see in **Figure 4** that the total loans percent changes are clustered near similar values regardless of metropolitan area. This shows the **trend is not due to regional effects near a particular city**.

```r
#Get total loans by Metropolitan Statistical Area/Metropolitan Division
dt.total.msamd =  dt.loans[,list(total = sum(loan.usd)), 
                                            by = list(year, MSA.MD, loan.purpose.desc)][order(total, decreasing = TRUE)]

#Get top 7 MSA.MDs
int.msamd.top <- unique(dt.total.msamd[,list(n = .N), by = list(MSA.MD, loan.purpose.desc)][n == 3 & !is.na(MSA.MD)]$MSA.MD)[1:7]
dt.total.msamd.top <- dt.total.msamd[MSA.MD %in% int.msamd.top]

#Look at percent change in loans by year
dt.total.msamd.top.2013 <- merge(dt.total.msamd.top[year == 2012], 
                                 dt.total.msamd.top[year == 2013], 
                                 by = c('MSA.MD', 'loan.purpose.desc'))[,pct_change := 100 * total.y / total.x]
dt.total.msamd.top.2014 <- merge(dt.total.msamd.top[year == 2013], 
                                 dt.total.msamd.top[year == 2014], 
                                 by = c('MSA.MD', 'loan.purpose.desc'))[,pct_change := 100 * total.y / total.x]

dt.total.msamd.top <- rbind(dt.total.msamd.top.2013, dt.total.msamd.top.2014)[,list(MSA.MD = MSA.MD, year = year.y, pct_change, loan.purpose.desc)]

#A point plot will best show the trend: all metroplitan areas had similar changes in total loans
ggplot(data=dt.map.msamd[dt.total.msamd.top], aes(x = as.character(year), y = pct_change, color = MSA.MD.desc, group = MSA.MD))+
  geom_point() + 
  labs(x = "Year)",
       y = "% Change in Total Loan From Previous Year",
       title = "Figure 4. % Total Loan Changes by MSA/MD")+ theme(legend.position="bottom", legend.direction="vertical", text = element_text(size=14), axis.text.x = element_text(angle=90)) + facet_wrap(~loan.purpose.desc)+ scale_color_discrete(name="Metropolitan Area")
```

<img src="/img/figure-4.png" width="700px"/>

#### Respondent
Next, we look at the change in loan amounts by loan company. It is possible that a change in lending behavior at a major loan company could be driving the trend. Looking at the respondents with the top 10 highest total home loan amounts, we see in **Figure 5** that there is **no single company exerting a major influence on the trend**. We are now confident that **the observed trend is general**, and is not being biased by factors in a particular region or by any particular loan company.

```r
#Get total loans by respondent
dt.total.respondent =  dt.loans[,list(total = sum(loan.usd)), 
                                                 by = list(year, respondent.name, loan.purpose.desc)][order(total, decreasing = TRUE)]

#Get top 7 MSA.MDs
chr.respondent.top <- unique(dt.total.respondent[,list(n = .N), by = list(respondent.name, loan.purpose.desc)][n == 3 & !is.na(respondent.name)]$respondent.name)[1:10]
dt.total.respondent.top <- dt.total.respondent[respondent.name %in% chr.respondent.top]

#Look at percent change in loans by year
dt.total.respondent.top.2013 <- merge(dt.total.respondent.top[year == 2012], 
                                      dt.total.respondent.top[year == 2013], 
                                      by = c('respondent.name', 'loan.purpose.desc'))[,pct_change := 100 * total.y / total.x]
dt.total.respondent.top.2014 <- merge(dt.total.respondent.top[year == 2013], 
                                      dt.total.respondent.top[year == 2014], 
                                      by = c('respondent.name', 'loan.purpose.desc'))[,pct_change := 100 * total.y / total.x]

dt.total.respondent.top <- rbind(dt.total.respondent.top.2013,dt.total.respondent.top.2013)[,list(respondent.name, year = year.y, pct_change, loan.purpose.desc)]

#A dot plot will best show the trend: all metroplitan areas had similar changes in total loans
ggplot(data=dt.total.respondent.top, aes(x = as.character(year), y = pct_change, color = respondent.name))+
  geom_point() + 
  labs(x = "Year)",
       y = "% Change in Total Loan From Previous Year",
       title = "Figure 5. % Total Loan Changes by Company")+ theme(legend.position="bottom", legend.direction="vertical", text = element_text(size=14), axis.text.x = element_text(angle=90)) + facet_wrap(~loan.purpose.desc)+ scale_color_discrete(name="Metropolitan Area")
```

<img src="/img/figure-5.png" width="700px"/>

### Lending Changes

We've shown the change in home loans were not limited by geographic area nor loan company. General, nationwide factors were responsible for the trend.

A fall in mortgage interest rates could explain the drop in refinancing. Home purchasing is more complicated, as it is largely driven by real estate prices, which is affected by more factors than the mortgage rate. That is largely outside of the scope of our dataset, but there is still interesting information we can extract from our data.

In the next sections, we'll look at some interesting trends which suggest **the fall in refinancing is in fact likely due to mortgage interest rates**, while the rise in home purchasing is likely due to a combination of a **change in lending standards** and **opportunistic buyers responding to lower home prices**.

#### Rise in Conventional Loans
One possible explanation for a rise in purchasing loans would be loosened lending standards. As a proxy for lending standard, we can look at the ratio of conventional to non-conventional loans. 

Home loans are made by private lenders, but can be backed by government agencies (non-conventional loan). These agencies, such as the Federal Housing Administration or Department of Veteran Affairs, insure non-conventional loans, allowing applicants to receive home loans who would not otherwise qualify. By contrast, conventional home loans are not backed by any government agency, and are typically characterized by stricter lending standards. A **rise in the proportion of conventional loans for equally qualified applicants therefore would indicate a loosening of lending standards**.

We see in **Figure 6** that while non-conventional home purchase loans stayed at similar levels throughout the three year period, conventional purchase loans increased. This shows that either **lending standards loosened for home purchase loans, or the number of highly qualified applicants increased**. We will investigate this further in the next section by looking only at low income individuals.

By contrast, refinancing loans fell in equal proportions, whether conventional or non-conventional. This shows **the fall in refinancing loans was driven by fewer people seeking refinancing loans** as opposed to a change in refinancing loan lending standards, supporting the idea that a drop in interest rates drove this change.

Note that we also looked at **conforming versus jumbo** loans, but did not find any obvious strong trends, so it is not included in this report. This is worth further investigation.

```r
dt.conventional =  dt.loans[,list(total = sum(loan.usd)/1e9), 
                                             by = list(year, status.conventional, loan.purpose.desc)][order(total, decreasing = TRUE)]

ggplot(data=dt.conventional, aes(x = year, y = total, color = status.conventional))+
  geom_line()+ geom_point() +
labs(x = "Year",
y = "Total Loan Amount (Billions)",
title = "Fig 6. Proportion of Purchase Conventional Loans Increased")+ theme(legend.position="bottom", text = element_text(size=14)) + facet_wrap(~loan.purpose.desc) + scale_color_discrete(name="Conventional Status")+ scale_x_continuous(breaks = c(2012:2014))

```
<img src="/img/figure-6.png" width="700px"/>

#### Rise in Conventional Loans for Low-Income Individuals
To determine whether lending standards loosened or the increase in conventional purchase loans was simply due to an increase in highly qualified applicants, we can look at total loans given to less qualified, low income individuals. We'll define low median income as being under 50k, though this threshold may be changed in the interactive plot below. We see in **Figure 7** that even among only low income indivudals, there was a shift from non-conventional to conventional loans. This confirms that **lending standards for home purchase loans loosened from 2012 to 2014**, partly explaining the increase in home purchase loans.

```r
#Interactive plot that allows threshold for low income to be changed
inputPanel(
sliderInput("low.income.threshold", label = "Low Income Threshold",
min = 10000, max = 150000, value = 50000, step = 10000)
)

renderPlot({
dt.conventional.low.income =  dt.loans[applicant.income.usd < input$low.income.threshold & loan.purpose.desc == 'Purchase',list(total = sum(loan.usd)/1e9),  by = list(year, status.conventional)][order(total, decreasing = TRUE)]

ggplot(data=dt.conventional.low.income, aes(x = year, y = total, color = status.conventional))+
  geom_line()+geom_point() + labs(x = "Year", y = "Total Loan Amount (Billions)",
title = "Figure 7. Low Income Purchase Loans")+ theme(text = element_text(size=14)) + scale_color_discrete(name="Conventional Status")+ scale_x_continuous(breaks = c(2012:2014))
}, width = 600)
```

<img src="/img/figure-7.png" width="700px"/>

#### Median Income Trends

There is more to the story that is revealed by looking at the reported median income of loan applicants. We can see in **Figure 8** that **high income individuals are disproportionately driving the change in home loans**. Note that since higher income individuals tend to receive larger loans, I chose to look at quantity of loans instead of total loan amount, which would otherwise have been a confounding factor in our data.

One plausible explanation is that high income indivuals are **more likely to take advantage of a fall in home prices by purchasing cheap real estate**. Additionally, the data shows these high income individuals are more likely than low income individuals to respond to changes in mortage interest rates by refinancing, since they were also disproportionately responsible for the fall in refinancing loans during this time period.

```r
#Make an interactive shiny plot showing change in median income of successful loan applicants
#Bar chart since we're using quantity
inputPanel(
sliderInput("n_breaks_income", label = "Number of bins:",
min = 2, max = 10, value = 4, step = 1)
)

renderPlot({
int.cuts = seq(from = 0, to = 120, length.out = as.integer(input$n_breaks_income))

dt.income.v.loan = dt.loans[!is.na(applicant.income.usd),list(year =year, loan.purpose.desc, bucket = cut2(applicant.income.usd/1000, cuts = int.cuts))]
dt.income.v.loan <- dt.income.v.loan[,list(count = .N), by = list(year, bucket, loan.purpose.desc)]
dt.income.v.loan <- dt.income.v.loan[,count_normalized := count / sum(count), by = list(year, loan.purpose.desc)]

ggplot(data=dt.income.v.loan, aes(x = bucket, y = count_normalized, fill = as.character(year))) +
geom_bar(stat='identity', position = 'dodge') +
labs(x = "Median Income (thousands)",
y = "Percentage of Loans",
title = "Figure 8. Median Income Disributions by Year")+ theme(text = element_text(size=14), axis.text.x = element_text(angle=0, vjust = 1, size = 9)) + scale_fill_discrete(name="Year")  +
facet_wrap( ~ loan.purpose.desc)
})
```

<img src="/img/figure-8.png" width="700px"/>

### Data Narrative: Summary of Key Insights
Based on our data, we saw two major trends from 2012 to 2014. Refinancing loans fell, while purchasing loans rose. These trends occurred nationwide and were not limited to any particular area or lending company. 

Refinancing loans did not decrease as a result of an increase in lending standards, suggesting they were instead driven by falling mortgage rates.

On the other hand, the increase in purchasing loans were driven by two factors. First, a loosening of lending standards contributed to an increase in home purchasing loans, which we showed by looking at the increasing proportion of conventional loans being given to low income applicants. Second, higher income individuals took out greater amounts of home purchasing loans. The easiest explanation for this phenomenon is that real estate prices fell, leading these individuals to capitalize on cheap real estate in anticipation of the market rebounding.

Based on these results, I **reject my initial hypothesis**. Despite the shrinking overall home loan market, **I recommend the following** (this is the final summary of our data narrative):

1. Our firm should **enter the home purchase loan product market**, which is increasing and now is a larger market than refinancing loans. Refinancing loans should be a lower priority unless we believe the mortgage rate will go up soon.
2. Our firm should **focus on VA and MD**, which together make up over 85% of the total market.
3. We can feel confident offering **conventional loans**, as the number of people seeking conventional loans is rising, and they make up the majority of all loans. 
4. Finally, our peer institutions have loosened their lending criteria; we should **re-evaluate our own lending criteria** and make sure they are not overly conservative in order to remain competitive.

I hope this helps out anyone trying to get started analyzing data with R. Feel free to leave a comment or question below.