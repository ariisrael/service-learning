# Service Learning

All of the US Department of Education and Data.gov APIs listed [here](https://usedgov.github.io/api/) are down. I [requested an API Key](https://api.data.gov/signup/) and tested each API endpoint with no success.

Previously, it looks like you could access:
- [College Scorecard](https://collegescorecard.ed.gov/data/documentation/)
- [My Brother's Keeper](https://usedgov.github.io/api/mbk.html)
- [Civil Rights Data Collection](https://usedgov.github.io/api/crdc.html)

It still lists them as if they worked, since there is a separate link for deprecated legacy APIs. None of them are active, with the last updates appearing to be in 2014 for My Brother's Keeper and the CRDC, and the College Scorecard returning a generic error page on the Data.gov website. None of this data can be accessed programmatically via API anymore. You can still download an [up-to-date College Scorecard dataset](https://collegescorecard.ed.gov/data/), but the [My Brother's Keeper downloadable datasets](https://www2.ed.gov/rschstat/statistics/surveys/mbk/index.html) have not been updated since 2014-2015.

The impact of this data not being accessible programatically is fewer data scientists or software engineers may be able to easily interact with the datasets. An API allows for more flexible usage of the data without needing to download and reformat CSVs or Excel Files into the right shape (changing columns and rows around) for data analysis. Some data analysts may not be comfortable using APIs and may prefer Excel files, so as long as the Excel file contains all the data for a period of time, I consider that to be good enough. 

----

I've decided to focus on the Civil Rights Data Collection (CRDC). Even when it was available via API, you could only access 2013-2014 data. However, the US Department of Education Office of Civil Rights (OCR) has maintained this dataset at irregular intervals, with the latest data being availabe for the 2017-2018 school year. More details about the CRDC can be found [here](https://www2.ed.gov/about/offices/list/ocr/data.html). The OCR portal for viewing and downloading the CRDC is [here](https://ocrdata.ed.gov/).

The data is searchable by district and by school. There does not appear to be any collated data source with all districts or all schools, which is unsurprising given the potential size of that dataset. Either way, in order to access this data programatically and make it availabe via API, I'll need either district or school names for the dataset year. I found a list of 2017-2018 school districts names and ids with county names/ids available on the [School Districts and Associated Counties page](https://www.census.gov/programs-surveys/saipe/guidance-geographies/districts-counties.html).

To start, I will focus on making this data available by district. Searching by school would be more difficult because the dataset would be larger and change more often. I also did not find a US Census file with school names by district, though I am sure it exists. There are still school districts with the same name, which is a complication I will need to address. Another complication I will have to consider is a school district located in multiple counties. 

The US Census does make available a 2017 [List of School Districts with the Same Name (by State)](https://www.census.gov/programs-surveys/saipe/guidance-geographies/same-name.2017.html), which should be helpful. However, it is in a table on an HTML page and not downloadable as a CSV, so there will be some work in translating that into a more usable data format. It also only provides school districts with the same name within a state, so if two school districts in two different states have the same name, they are not listed. For example, there is a Birmingham City School District in both Michigan and Alabama. Specifying the State when searching for the district should alleviate this problem.

My goal will be to make the most recent year's CRDC data by school district available via API. Right now, you have to query each school district individually or download the entire dataset. When I downloaded the entire dataset, the fields were confusing and though it contained every school district, it did not seem to include all the data available via the web interface. Via my API, you will be able to query by district or by state and without filters to see the entire dataset with readable and complete fields.

I pasted the [List of School Districts with the Same Name (by State)](https://www.census.gov/programs-surveys/saipe/guidance-geographies/same-name.2017.html) into a spreadsheet and cleaned the "COUNTY(s) SCHOOL DISTRICT LOCATED" column by splitting each county into its own column since there are a maximum of three counties per school district. I also cleaned the data downloaded from [School Districts and Associated Counties page](https://www.census.gov/programs-surveys/saipe/guidance-geographies/districts-counties.html) for the 2017 reporting year (2017-2018 school year). For both Census datasets, I renamed the column headers and saved the files as CSVs to store in this repository within the `/data/census/2017` folder. The `district_id` does not correspond to the NCES district id which you can use to search the OCR database. Ideally, I will be able to find a data set with the NCES district ids because searching on names and states feels brittle.

---

In my research, I ended up finding an API that I think has all the data I am looking for in one place. The [Urban Institute](https://www.urban.org/) has a [public API](https://educationdata.urban.org/documentation/#about_the_api) and [data explorer](https://educationdata.urban.org/data-explorer/) tool that collates data on schools, districts, and colleges from the Common Core of Data, Civil Rights Data Collection, Small Area Income and Poverty Estimates program, EDFacts, the NCES Integrated Postsecondary Education Data System, the National Historical Geographic Information System, and the Office of Federal Student Aid (see [data sources](https://educationdata.urban.org/documentation/#data_sources) page).

The Urban Institute Education Data API has packages for R and Stata but not for JavaScript or Python, where they suggest [directly accessing](https://educationdata.urban.org/documentation/#how_to_use) the URLs using the language's respective HTTP request library. I have decided my new goal for this project will be to publish a Python package to the public [Python Package Index (PyPI)](https://pypi.org/) that will wrap the Urban Institute Education Data API calls in easy-to-use functions. 

Currently, working with the API in Python might look like this:
```python
from urllib import urlopen
from json import loads

BASE_URL = "https://educationdata.urban.org/api/v1"
topic = "schools" # schools, school-districts, colleges
source = "ccd" # depends on topic
endpoint = "enrollment" # depends on source
year = 2012
specifier = 'grade-5' # depends on endpoint
disaggregator = 'sex' # depends on endpoint
url = BASE_URL + "/{topic}/{source}/{endpoint}/{year}/{specifier}/{disaggregator}"
url = url.format(
  topic=topic, 
  source=source, 
  endpoint=endpoint, 
  year=year, 
  specifier=specifier, 
  disaggregator=disaggregator)
response = urlopen(url)
data = loads(response.read())
```
This request would get enrollment data for all schools with disaggregated enrollment by sex. The developer would have to either hard code the URLs or build them dynamically as above, both of which would be time consuming and would not follow the DRY programming principle (Don't Repeat Yourself). 

Here is an idea for improving the workflow for the same request using my python wrapper. For the example, I'll pretend we've named the package "uied" for (U)rban (I)nstitute (E)ducation (D)ata. First the developer would install the package via the command line, and then they would import the package to use in their own script. 
```
$ pip install uied
```
```python
import uied

data = uied.schools.ccd.enrollment.year(2015).grade(5).sex().get()
```
---
This will be the first package I am publishing in Python, so I've been consulting resources like [How to Publish an Open-Source Python Package to PyPI](https://realpython.com/pypi-publish-python-package/) and [Packaging Python Projects](https://packaging.python.org/tutorials/packaging-projects/). I am going to start by wrapping the [school-districts topic](https://educationdata.urban.org/documentation/school-districts.html) since my original goal was to make working with district-level data easier.

## [School Districts](https://educationdata.urban.org/documentation/school-districts.html)

### **Sources**
- [Common Core of Data](https://nces.ed.gov/ccd/): `/ccd`
- [Small Area Income and Poverty Estimates](https://www.census.gov/programs-surveys/saipe.html): `/saipe`
- [EDFacts](https://www2.ed.gov/about/inits/ed/edfacts/index.html): `/edfacts`

### `/edfacts`
- State Assessments: `/assessments`
  - Required: year and grade

### `/saipe`
- Poverty Assessments: `/`
  - Required: year

### `/ccd`
- Directory: `/directory`
  - Required: year
- Enrollment: `/enrollment`
  - Required: year and grade
- Finance: `/finance`
  - Required: year

There are also additional filter parameters for each endpoint that I will deal with later. Right now, my priority will be to wrap the three school district source endpoints and set up the basic package structure and syntax.

---
## Appendix 
### Existing Data Tools
- [NAEP data explorer](https://www.nationsreportcard.gov/ndecore/landing)
- [List of NCES data tools](https://nces.ed.gov/datatools/)
- [Search data.gov datasets](https://catalog.data.gov/dataset)
- [College scorecard](https://collegescorecard.ed.gov/)
- [My Brother's Keeper datasets](https://www2.ed.gov/rschstat/statistics/surveys/mbk/index.html) - not updated
- [Department of Education data APIs](https://usedgov.github.io/api/) - not working
- [OCR search for districts](https://ocrdata.ed.gov/flex/Reports.aspx?type=district)
- [OCR search for schools](https://ocrdata.ed.gov/flex/Reports.aspx?type=school)
- [Census SAIPE data](https://www.census.gov/programs-surveys/saipe/data.html)
- [Urban Institute data explorer](https://educationdata.urban.org/data-explorer/)

### Government Sources
- U.S. Department of Education
  - [Office of Civil Rights](https://www2.ed.gov/about/offices/list/ocr/index.html)
  - [Institute of Education Sciences](https://ies.ed.gov/)
    - [National Center for Education Statistics](https://nces.ed.gov/)
      - [National Assessment of Educational Progress](https://nces.ed.gov/nationsreportcard/)
      - [Common Core of Data](https://nces.ed.gov/ccd/search.asp)
- U.S. Census Bureau
  - [Small Area Income and Poverty Estimates Program](https://www.census.gov/programs-surveys/saipe.html)
- U.S. General Services Administration
  - [Technology Transformation Services](https://www.gsa.gov/about-us/organization/federal-acquisition-service/technology-transformation-services)
    - [data.gov](https://www.data.gov/)

### Abbreviations
- **LEA**: Local educational agency
- **NCES**: National Center for Education Statistics
- **NAEP**: National Assessment of Educational Progress
- **CRDC**: Civil Rights Data Collection
- **OCR**: Office of Civil Rights
- **IES**: Institute of Education Sciences
- **SAIPE**: Small Area Income and Poverty Estimates Program
- **MBK**: My Brother's Keeper
- **CCD**: Common Core of Data













