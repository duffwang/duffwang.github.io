---
layout: post
title: Tutorial on Data Cleaning and Transformation using R
---

To demonstrate proper data cleaning (also known as data munging) and transfomation, we will be working with a [publically available](https://www.ffiec.gov/hmda/hmdaflat.htm) dataset on Home Mortgage Loans. 

All the code I wrote to clean/transform the data in R is shown alongside each section.

## Data Description

The dataset was generated in compliance with the Home Mortgage Disclosure Act of 1975, which requires disclosures on the originations and purchases of home purchase, improvement, and refinancing loans. The 2012-2014 data used in this report has been filtered to five states (MD, VA, WV, DC, DE) and to 1-4 family, owner-occupied home loans, with an originated action type, secured by a first or subordinate lien.

### Reading in Data

The first step is to read in the raw data files from the link above. To do this, I write an R function that parses the data, does basic cleanup steps (as shown below), and also is memoized (saving a cached version of the dataset for faster future loading times). The code is shown with comments explaining what I am doing.

```r
###INSTRUCTIONS### 
#1. Install these packages if necessary
#install.packages(c("data.table", "Hmisc", "reshape2", "ggplot2", "shiny", "jsonlite", "stringdist"))
#2. Set the working directory to the folder containing the data files and the two csv data files
#setwd("PATH")
#3. Run the document

#This function reads the data files and returns an expanded HDMA data table
#It uses a caching system to save the data tables to compressed RDS format, so it can be loaded more quickly next time.
hmda_init <- function(chr.file.loans = "2012_to_2014_loans_data", chr.file.institutions = "2012_to_2014_institutions_data") {
  
  #Since these are large csvs, we'd like to cache the results so we don't need to parse them in every time
  #This function reads a file (chr.filename).csv and returns a data table, caching results for speed
  read.csv.memoized <- function(chr.filename) {
    chr.filename.cached <- paste0(chr.filename, "_cached.rds") 
    if (file.exists( chr.filename.cached )) {
      return(readRDS(chr.filename.cached))
    } else {
      if (file.exists(paste0(chr.filename, ".csv"))) {
        dt.data <- data.table( read.csv( paste0(chr.filename, ".csv"), stringsAsFactors = FALSE ) )
        #@todo: we could optimize memory usage and speed up our code by using factors instead of strings
        saveRDS(dt.data, chr.filename.cached)
        return(dt.data)
      } else {
        stop("Couldnt find the csv data files. Please add them to the directory")
      }
    }
  }
  
  #Read input data
  dt.loans <- read.csv.memoized(chr.file.loans)
  dt.institutions <- read.csv.memoized(chr.file.institutions)
  
  #Next, we clean up the data.
  
  #Standardize the column names, and ensure numeric data types are correct
  #For speed reasons, we won't convert to mixed case on the character vectors until needed, since that is an expensive operation
  dt.loans <- dt.loans[,list(year = as.numeric.na(As_of_Year, bool.integer = TRUE),
                             code.agency = as.numeric.na(Agency_Code, bool.integer = TRUE),
                             code.agency.desc = Agency_Code_Description,
                             id.respondent = Respondent_ID, #this column is truly alphanumeric, leave as character
                             sequence.number = as.numeric.na(Sequence_Number, bool.integer = TRUE),
                             loan.usd = as.numeric.na(Loan_Amount_000, bool.integer = TRUE) * 1000,
                             applicant.income.usd = as.numeric.na(Applicant_Income_000, bool.integer = TRUE) * 1000,
                             loan.purpose.desc = Loan_Purpose_Description,
                             loan.type.desc = Loan_Type_Description,
                             lien.status.desc = Lien_Status_Description,
                             code.status = as.numeric.na(State_Code, bool.integer = TRUE),
                             state.short = State,
                             code.county = as.numeric.na(County_Code, bool.integer = TRUE),
                             MSA.MD = as.numeric.na(MSA_MD, bool.integer = TRUE),
                             MSA.MD.desc = MSA_MD_Description,
                             tract.number = as.numeric.na(Census_Tract_Number),
                             MSA.MD.median.income = as.numeric.na(FFIEC_Median_Family_Income, bool.integer = TRUE),
                             tract.to.MSA.MD.median.income.pct = as.numeric.na(Tract_to_MSA_MD_Income_Pct), #Treat blanks as NA
                             tract.units = as.numeric.na(Number_of_Owner_Occupied_Units, bool.integer = TRUE),
                             county.name = County_Name,
                             conforming.limit = as.numeric.na(Conforming_Limit_000, bool.integer = TRUE) * 1000,
                             status.conventional = Conventional_Status,
                             status.conforming = Conforming_Status,
                             status.conventional.conforming = Conventional_Conforming_Flag
  )]
  
  
  dt.institutions <- dt.institutions[,list(year = as.numeric.na(As_of_Year, bool.integer = TRUE),
                                           code.agency = as.numeric.na(Agency_Code, bool.integer = TRUE),
                                           id.respondent = Respondent_ID,
                                           respondent.name = HmdaMixedCase(Respondent_Name_TS),
                                           respondent.city = HmdaMixedCase(Respondent_City_TS),
                                           respondent.state = Respondent_State_TS,
                                           respondent.zip = Respondent_ZIP_Code,
                                           parent.name = HmdaMixedCase(Parent_Name_TS),
                                           parent.city = HmdaMixedCase(Parent_City_TS),
                                           parent.state = Parent_State_TS,
                                           parent.zip = Parent_ZIP_Code,
                                           assets.panel = as.numeric.na(Assets_000_Panel) * 1000 #using numeric because values exceeds int max value (2e9)
  )]
  
  setkey(dt.institutions, year, id.respondent, code.agency)
  setkey(dt.loans, year, id.respondent, code.agency, sequence.number)
  
  #Check data for primary key issues
  if (nrow(dt.loans) != 
      nrow(unique(dt.loans[,list(year, id.respondent, code.agency, sequence.number)]))) {
    warning("Duplicate year, respondent id, agency code, sequence number  entries in institutions data file.")
  }
  if (nrow(dt.institutions) != 
      nrow(unique(dt.institutions[,list(year, id.respondent, code.agency)]))) {
    warning("Duplicate year, respondent id, agency code entries in institutions data file.")
  }
  
  #We'll merge the two data tables
  #I will keep only respondent name from the institutions table as that is my interpretation of the instructions, and otherwise
  #the dataset will be too wide for easy use for the user, containing mostly useless info about the respondents.
  #We'll also create a new attribute that buckets loan amount. I'll bucket it into 10 groups, each with roughly equal number.
  dt.loans <- dt.institutions[,list(year, code.agency, id.respondent, respondent.name)][dt.loans]
  dt.loans <- dt.loans[, loan.usd.bucket := cut2(loan.usd, g = 10)]
  
  setkey(dt.loans, year, id.respondent, code.agency, sequence.number)

  return(dt.loans)
}

#' This function takes the data table returned by hdma_init and exports it as a JSON file
#'
#' @param data data.table containing loan and institution information
#' @param states (optional) character vector of 2 letter state abbreviations to filter by
#' @param conventional_conforming (optional) logical TRUE will filter loans to conventional, conforming loans. FALSE returns all other loans. Leave as NA to include all loans.
#' @param chr.output.file Filename of the exported JSON file, written with ".json" file extension. Default is "hdma"
#' 
#' @return \code{TRUE}
#'
#' @author Duff Wang
#'
hmda_to_json <- function(data, states = NA, conventional_conforming = NA, chr.output.file = "hdma") {
  #Input checks
  if (!any(grepl("data.table", class(data)))) { stop("Expecting a data.table. Check your input data") }
  dt.input <- data 
  
  #Ensures states is in right format if provided
  if (length(states) == 1 && !is.na(states)) {
    if (class(states) != "character") { stop("states must be a character vector") }
    if (any(nchar(states) != 2)) { stop("States must contain two character abbreviations") }
  }
  
  #Ensures conventional_conforming is in right format if provided
  if (length(conventional_conforming) > 1) { stop("conventional_conforming must be a single value") }
  if (!is.na(conventional_conforming) && class(conventional_conforming) != "logical") {
     stop("conventional_conforming must be a logical vector") 
  }

  #Subset the data accordingly
  if (length(states) != 1 || !is.na(states)) {
    dt.input <- dt.input[state.short %chin% states]  #%chin% is much faster than %in%
  }
  if (!is.na(conventional_conforming)) {
    dt.input <- dt.input[(status.conventional.conforming == 'Y') == conventional_conforming] 
  }
  
  #Export JSON object
  json.output <- toJSON(dt.input)
  write(json.output, paste0(chr.output.file, ".json"))
  
  return(TRUE)
}
```

Now we simply need to call our data parsing function.

```r
#Now, we import the raw data.
#Read input data
dt.loans <- hmda_init()
```

### Data Sanity Check: Loan Amount
First, we turn out attention to the loan amount column. We want to look for any spurious data points. The magnitude of loan amounts may vary for different properties, but they should not be drastically more than the median income of the applicant. We will look at each respondent and calculate their average loan to income ratio. 

One notable outlier comes from **Megachange Financial**, a company that gave out loans with loan to median income ratios that were three orders of magnitude larger than any other lender, as shown in **Figure 1** with the outlier highlighted in red. The average loan to income ratio for this firm was around 1000, while the second highest was at only around 10. Megachange loans are in the range of tens of millions of dollars, with typical median income less than $100k.

One possible explanation is that the firm incorrectly submitted the loan amounts in units of dollars instead of units of thousands of dollars, which is what the dataset uses for that field. This would explain the order of magnitude disprepancy being three. Another explanation is that these are atypical loans given for extremely high value properties, perhaps given to high net worth individuals. 

As these loans account for less than **0.1%** of all loans in our dataset, are possibly the result of a mistake, and fail our sanity check, I have chosen to remove these outliers from our data set. 

As a future quality check rule, I would implement a check for any respondent with an average loan to income **ratio greater than 20**. I chose this threshold because our dataset shows every other firm has such a ratio firmly below 20, and that threshold should capture any obvious outliers such as Megachange Mortgage.

```r
setkey(dt.loans, applicant.income.usd, year, code.agency, id.respondent, sequence.number)

#Find the highest loan / income ratios
dt.loans <- dt.loans[,loan.income.ratio := loan.usd / applicant.income.usd]
dt.loans <- dt.loans[applicant.income.usd == 0L, loan.income.ratio := NA_integer_]

setkey(dt.loans, id.respondent, code.agency, year, sequence.number)
dt.highest.loan.income.ratio <- dt.loans[!is.na(loan.income.ratio),list(loan.income.ratio.avg = mean(loan.income.ratio)), by = list(id.respondent, code.agency, respondent.name)]

dt.highest.loan.income.ratio <- dt.highest.loan.income.ratio[order(loan.income.ratio.avg, decreasing = T)]
dt.highest.loan.income.ratio <- dt.highest.loan.income.ratio[, respondent.name := factor(respondent.name, levels = unique(respondent.name))] #Reorder factor levels so stacked bar chart is ordered how we want


#Show top 10 average loan to income ratios by respondent, which shows the outlier
ggplot(data=dt.highest.loan.income.ratio[1:10], aes(x = respondent.name, y = loan.income.ratio.avg))+
  geom_point() + 
  labs(x = "Respondent",
       y = "Average Loan to Median Income Ratio",
       title = "Figure 1. Top Ten Average Loan to Median Income Ratios")+ theme(text = element_text(size=14), axis.text.x = element_text(angle=45, vjust = 1, hjust = 1))+ annotate("rect", xmin=0.6, xmax=1.4, ymin=-50, ymax= 1050, fill=NA, colour="red")

#Remove outlier data point
dt.loans <- dt.loans[code.agency != 9 | id.respondent != 9731400737]

setkey(dt.loans, year, id.respondent, code.agency, sequence.number)
```
<img src="/img/figure-1.png" width="700px"/>

### Data Mapping: Respondent Name

Next, the respondent name field suffers from typos or slight formatting changes, causing a single respondent to be listed multiple times under different names.

The best way to resolve this is to create a concept of "**respondent name anchors**". These respondent anchors correspond one to one with actual respondents. We will then proceed to try various strategies to "map" the respondent names provided to these unique respondent name anchors.

One good way to do this is by measuring each respondent name's **Levenstein distance** with every other respondent name. A Levenstein distance assigns a penalty for each deletion, insertion, and substitution that is required to transform one string into another. We then use a cutoff Levenstein distance to form a map of respondent names that are very close to another respondent name. We'll assign double penalty to substitution as opposed to deletion or insertion, since most typos will be in the form of an extra space or comma, while entire character changes suggest they are different banks with similar names.

One issue with this strategy is that many banks use abbreviations that could be only off by a letter. We will implement **another rule** that states if the first three letters of any two names are different, then they are different entities. This should remove most of these false positives while still keeping most of the truly needed mappings.

The results of our mapping is shown in the table below. We found fixed **84** respondent names out of **1,602**. As you can see by scrolling through the table, our conservative mapping rules produced no false positives. Using the 'respondent name anchor' concept, we can continue to add new rules and build more filters to "map" respondent names. As it grows more complex, however, we will need more manual checking and failsafe logic, so we'll keep just the two simple rules described for now.

In the future, I would use the rules implemented above, as well as **propose a mapping system** with looser criteria where the end user can manually choose to map new data, after being presented with a list of the closest Levenstein distance matches.

```r
#We will use a levenstein survey, with a higher penalty for substitutions, as those make it more likely it is two different banks with similar names.
#We're mostly looking for deletion and insertion typos.
chr.respondent.unique <- unique(dt.loans$respondent.name)
mat.lv.distance <- stringdistmatrix(chr.respondent.unique, chr.respondent.unique, method = "lv", weight = c(0.5, 0.5, 1, 1))
mat.lv.distance[mat.lv.distance == 0] = Inf #Exclude "from" strings from matching with themselves, as we want the next closest match
int.lv.min <- apply(mat.lv.distance,2,function(x) return(array(min(x)))) #Get closest match for each respondent
int.map.from <- which(int.lv.min <= 1) #This threshold allows 2 deletions/insertions, and 1 substitution
int.map.to <- apply(mat.lv.distance[,int.map.from], 2, function(x) return(which.min(x))) #Retrieve which index to map to
dt.map.name = data.table(from = seq_along(chr.respondent.unique), to = seq_along(chr.respondent.unique)) #Create a mapping table
dt.map.name[int.map.from]$to = pmin(int.map.to, dt.map.name[int.map.from]$to) #We'll always soon the first respondent as our 'anchor' respondent we map to
dt.map.name <- dt.map.name[,from_name := chr.respondent.unique[from]]
dt.map.name <- dt.map.name[,to_name := chr.respondent.unique[to]]

#Because many banks use abbreviations that are only a character off, let's filter out anything where the first three letters don't match
invisible(dt.map.name[substring(from_name, 1, 3) != substring(to_name, 1, 3), to := from])
setkey(dt.map.name, from_name)

dt.loans <- merge(dt.loans, dt.map.name[,list(respondent.name = from_name, respondent.name.anchor = to_name)], by = "respondent.name")

setkey(dt.loans, year, id.respondent, code.agency, sequence.number)
```

<div style="height:400px;overflow:auto">
<p><em>This table shows only entries where a respondent name was mapped to a different anchor. All other entries were mapped to themselves.</em></p>
<pre><code>##                      Original Name                    Anchor Name
##  1:    1st Colonial Community Bank   1st Colonial Community  Bank
##  2:    1st Portfolio Lending Corp.     1st Portfolio Lending Corp
##  3:   Aberdeedn Proving Ground Fcu    Aberdeen Proving Ground Fcu
##  4:   Advisors Mortgage Group, Llc  Advisors Mortgage Group, Llc.
##  5:      Agchoice Farm Credit, Aca    Agchoice Farm Credit,   Aca
##  6:  American Equity Mortgage, Inc American Equity Mortgage, Inc.
##  7:  American Financial Resources,   American Financial Resources
##  8:      Appalachain Community Fcu      Appalachian Community Fcu
##  9: Associated Mortgage Bankers, I Associated Mortgage Bankers In
## 10:   Atlantic Coast Mortgage, Llc    Atlantic Coast Mortgage Llc
## 11:   Aurora Financial Group, Inc.     Aurora Financial Group Inc
## 12:  Baltimore County Empoyees Fcu Baltimore County Employees Fcu
## 13:                Bny Mellon N.a.               Bny Mellon, N.a.
## 14:           Broker Solutions Inc         Broker Solutions, Inc.
## 15:   Bulldog Federal Credit Union   Bullldog Federa Credit Union
## 16:   C.u. Mortgage Services, Inc.    C.u. Mortgage Services, Inc
## 17:      Calhoun County Bank, Inc.       Calhoun County Bank, Inc
## 18:                Capital One, Na                 Capital One Na
## 19:    Cis Financial Services Inc.   Cis Financial Services, Inc.
## 20:              Citizens Bank, Na            Citizens Bank, N.a.
## 21:         Colonial Virginia Bank          Colonial Virgnia Bank
## 22:      Community Trust Bank Inc.     Community Trust Bank, Inc.
## 23:  Corridor Mortgage Group, Inc.   Corridor Mortgage Group, Inc
## 24:        Crossline Capital, Inc.         Crossline Capital Inc.
## 25:       Damascus Community  Bank        Damascus Community Bank
## 26: Db Private Wealth Mortgage Ltd Db Private Weath Mortgage Ltd.
## 27:       Discover Home Loans, Inc      Discover Home Loans, Inc.
## 28: Doolin Security Savings Bank F   Doolin Security Savings Bank
## 29:    E Mortgage Manamgement, Llc     E Mortgage Management, Llc
## 30:       Eastern Savings Bank Fsb      Eastern Savings Bank, Fsb
## 31:  Fairway Independent Mort Corp Fairway Independent Mort. Corp
## 32:   Farm Credit Of The Virginias  Farm Credit Of The Virginias,
## 33:          Fearon Financial, Llc           Fearon Financial Llc
## 34:   First Eagle Federal Credit U First Eagle Federal Credit Uni
## 35:                  Fnb Bank, Inc                 Fnb Bank, Inc.
## 36:  Franlkin American Mortgage Co  Franklin American Mortgage Co
## 37:          Fredreick County Bank          Frederick County Bank
## 38:                Fulton Bank, Na              Fulton Bank, N.a.
## 39: Glen Burnie Mutual Savngs Bank Glen Burnie Mutual Savings Bnk
## 40:                       Gmfs Llc                      Gmfs, Llc
## 41:                Grand Bank N.a.                  Grand Bank Na
## 42:       Guardhill Financial Corp      Guardhill Financial Corp.
## 43:    Hamilton Group Funding, Inc   Hamilton Group Funding, Inc.
## 44:         Hartford Funding, Ltd.          Hartford Funding Ltd.
## 45:       Homeward Residential Inc      Homeward Residential, Inc
## 46:             Iab Financial Bank              Iab Finacial Bank
## 47: Intgrity First Financial Group Integrity First Financial Grou
## 48:   Johnson Mortgage Company Llc  Johnson Mortgage Company, Llc
## 49:               Loan Simple Inc.              Loan Simple, Inc.
## 50:                   Milend, Inc.                    Milend, Inc
## 51: Morgan Stanley Private Bank, N Morgan Stanley Private Bank Na
## 52:          Mortgage America, Inc         Mortgage America, Inc.
## 53:         Mortgage Assurance Inc        Mortgage Assurance Inc.
## 54:                 Mvb Bank, Inc.                   Mvb Bank Inc
## 55:    Nations Direct Mortgage Llc   Nations Direct Mortgage, Llc
## 56:          New Horizon Bank N.a.            New Horizon Bank Na
## 57:       Oak Mortgage Company Llc     Oak Mortgage Company , Llc
## 58:        Old Point Mortgage, Llc         Old Point Mortgage Llc
## 59:             On Q Financial Inc           On Q Financial, Inc.
## 60:       Peoples Home Equity, Inc        Peoples Home Equity Inc
## 61:   Peoplesbank A Codorus Valley  Peoplesbank, A Codorus Valley
## 62:    Platinum Home Mortgage Corp     Platinum Home Mortage Corp
## 63:    Pmac Lending Services, Inc.     Pmac Lending Services, Inc
## 64:              Premier Bank Inc.               Premier Bank Inc
## 65:    Premier Home Mortgage, Inc.      Premier Home Mortgage Inc
## 66:   Prime Mortgage Lending, Inc.    Prime Mortgage Lending Inc.
## 67:                   Resmac, Inc.                    Resmac, Inc
## 68:        Severn Savings Bank Fsb       Severn Savings Bank, Fsb
## 69:           Sirva Mortgage, Inc.            Sirva Mortgage, Inc
## 70:         Sperry Associaties Fcu          Sperry Associates Fcu
## 71:   State Employees Credit Union  State Employees' Credit Union
## 72:            Summit Funding, Inc           Summit Funding, Inc.
## 73:             Suntrust Bank, Inc            Suntrust Banks, Inc
## 74:          The Money Source Inc.           The Money Source Inc
## 75:   Total Mortgage Services, Llc    Total Mortgage Service, Llc
## 76:  Truliant Federal Credit Union Truliant Federal Credit  Union
## 77:                    Umb Bank Na                   Umb Bank, Na
## 78:       Union Home Mortgage Corp      Union Home Mortgage Corp.
## 79:          United Mortgage Corp.           United Mortgage Corp
## 80:     Urban Financial Group, Inc     Urban Financial Group Inc.
## 81:      Waterstone Mortgage Corp.       Waterstone Mortgage Corp
## 82:         Weststar Mortgage, Inc          Weststar Mortgage Inc
## 83:        Weststar Mortgage, Inc.         Weststar Mortgage, Inc
##                      Original Name                    Anchor Name</code></pre>
</div>



### Data Mapping: Metropolitan Area
Finally, we do a data quality check on metropolitan area. Firstly, everything is in uppercase, but we would prefer to get it in a normal mixed case format. Because the entries are in address format (e.g. **Richmond, VA**) we write a custom function to convert these names to mixed case.

It would take a long time to run our function on every single row in the table, as the case changing function does not natively take advantage of R's 'vectorized' functionality. Instead, we'll create a **separate mapping table**, going from MSA/MD id to a cleaned up version of the name.

An initial look at the metropolitan areas reveals many entries that are very similar, differing only by a city name or state, or in some different order. We could potentially employ the same strategy as for the respondent names, where we map together similar metropolitan areas. However, a look at the [FFIEC website](https://www.ffiec.gov/geocode/help1.aspx) reveals these **metropolitan areas are adjusted annually**. Since the boundaries are changing, we would not want to map these metropolitan areas together, as they wouldn't actually be the same entity. 

We'll leave these as is, but depending on what we need for our data analysis (to be discussed in a future post), we could potentially link together similar MSA/MD entities.

```r
#Create useful mapping table
dt.map.msamd <- unique(dt.loans[,list(MSA.MD, MSA.MD.desc.old = MSA.MD.desc)])[!is.na(MSA.MD)] #msa/md id to description
dt.map.msamd <- dt.map.msamd[,MSA.MD.desc := HmdaMixedCaseAddress(MSA.MD.desc.old)]
setkey(dt.map.msamd, MSA.MD)
```

<div style="height:400px;overflow:auto">
<p><em>Note: entries are trimmed to max 30 characters, for formatting purposes.</em></p>
<pre><code>##                      Original Name                     Mixed Case
##  1:           BALTIMORE-TOWSON, MD           Baltimore-Towson, MD
##  2:  BALTIMORE-COLUMBIA-TOWSON, MD  Baltimore-Columbia-Towson, MD
##  3:                    BECKLEY, WV                    Beckley, WV
##  4: BETHESDA-ROCKVILLE-FREDERICK,  Bethesda-Rockville-Frederick, 
##  5: BLACKSBURG-CHRISTIANSBURG-RADF Blacksburg-Christiansburg-Radf
##  6:  CALIFORNIA-LEXINGTON PARK, MD  California-Lexington Park, MD
##  7:                 CHARLESTON, WV                 Charleston, WV
##  8:            CHARLOTTESVILLE, VA            Charlottesville, VA
##  9:              CUMBERLAND, MD-WV              Cumberland, MD-WV
## 10:                   DANVILLE, VA                   Danville, VA
## 11:                      DOVER, DE                      Dover, DE
## 12:  HAGERSTOWN-MARTINSBURG, MD-WV  Hagerstown-Martinsburg, MD-WV
## 13:               HARRISONBURG, VA               Harrisonburg, VA
## 14:   HUNTINGTON-ASHLAND, WV-KY-OH   Huntington-Ashland, WV-KY-OH
## 15: KINGSPORT-BRISTOL-BRISTOL, TN- Kingsport-Bristol-Bristol, TN-
## 16:                  LYNCHBURG, VA                  Lynchburg, VA
## 17:                 MORGANTOWN, WV                 Morgantown, WV
## 18: PARKERSBURG-MARIETTA-VIENNA, W Parkersburg-Marietta-Vienna, W
## 19:         PARKERSBURG-VIENNA, WV         Parkersburg-Vienna, WV
## 20:                   RICHMOND, VA                   Richmond, VA
## 21:                    ROANOKE, VA                    Roanoke, VA
## 22:                  SALISBURY, MD                  Salisbury, MD
## 23:               SALISBURY, MD-DE               Salisbury, MD-DE
## 24: SILVER SPRING-FREDERICK-ROCKVI Silver Spring-Frederick-Rockvi
## 25:        STAUNTON-WAYNESBORO, VA        Staunton-Waynesboro, VA
## 26:    STEUBENVILLE-WEIRTON, OH-WV    Steubenville-Weirton, OH-WV
## 27: VIRGINIA BEACH-NORFOLK-NEWPORT Virginia Beach-Norfolk-Newport
## 28: WASHINGTON-ARLINGTON-ALEXANDRI Washington-Arlington-Alexandri
## 29:    WEIRTON-STEUBENVILLE, WV-OH    Weirton-Steubenville, WV-OH
## 30:                WHEELING, WV-OH                Wheeling, WV-OH
## 31:           WILMINGTON, DE-MD-NJ           Wilmington, DE-MD-NJ
## 32:              WINCHESTER, VA-WV              Winchester, VA-WV
##                      Original Name                     Mixed Case</code></pre>
</div>

### Remarks

I hope this tutorial helped give you some understanding of the challenges faced when presented with a new dataset. The data may contain outliers, have typos, and have different formats or representations of the same underlying data. Always make sure to sanity check your data before moving on to analysis.
