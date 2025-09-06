# Canadian Postsecondary Education Alcohol and Drug Use Analysis

## Overview
This project analyzes the Canadian Postsecondary Alcohol and Drug Survey (CPADS) 2021-2022 Public Use Microdata File (PUMF), which contains responses from 40,931 post-secondary students across Canada on substance use, health, and demographics. The dataset with 394 variables was processed to extract key insights on alcohol, cannabis, and vaping use, segmented by demographics like gender, ethnicity, and age. The project demonstrates end-to-end data engineering and analysis skills, including data extraction, cleaning, database management, and visualization.

### Key technologies used:

- Python: Initial data exploration and PDF parsing attempts.
- SQL Server Management Studio (SSMS): Data loading, schema design, and view creation.
- Power BI: Data cleaning, transformation, and interactive visualizations.

The final output is a Power BI dashboard with visualizations to explore substance use prevalence, frequency, and correlations with mental/physical health.


## Steps

### Data Acquisition

Obtained the CPADS 2021-2022 PUMF dataset (CSV) and accompanying explanatory guide and codebook (PDF) containing variable labels and codes.
Explored dataset in Python (Pandas) to understand structure. 

### Data Exploration

Decided to use Python (Pandas) for quick preliminary data analysis as the survey count was over 40,000 students, which could cause preformance inssues in Excel.
The following code outputst the contents of the file in a table displaying the basic structure.

```
import pandas as pd
df = pd.read_csv('filepath')
df
```

The file contained 40,931 rows and 394 columns. It was structued in a way that each column corresponded to a survey question and each categorical response was then assigned a number and entered into the row. Viewing the PDF guidebook I could match each column name to a column and each row value to a response.
For example in the guidebook the first question

<img width="1067" height="172" alt="image" src="https://github.com/user-attachments/assets/22b4ea16-c4ab-4104-aead-c4f039f09fd6" />

Within the responses CSV file that data would be recorded as

| stu04  |
| ------------- |
| 1  |
| 2  |
| 1  |
| ... |

## Analysis Plan

Knowing the structure of the data, a plan can be formulated. Due to the data file only containing numbers and codes, at some point the corresponding labels from the PDF will have to be matched up to the response values. If the table from the PDF can be extracted as text in a tabular format, a lookup can then be executed. 

For the purpose of this project, a SQL database will be used (Microsoft Sequel Server Management Studio) to host the data and complete the data lookups (Joining). Completing joins is much faster within SQL than the alternative option of merging within Power BI. Another benefit is that we can maintain the origional table by utilizing a view to complete the joining with only our needed columns. 
Another alternative could be to create a STAR schema by creating seperate dimension tables for each question and connecting them via relationsips within Power BI but this could cause scaling issues if more variables were ever in need of analysis due to a new table required for each variable.


### Codebook Extraction

Attempted to parse the codebook PDF using standard PDF tools and Python libraries (PyPDF2), but faced formatting issues due to inconsistent PDF headings. With the data available but needing to be reformatted, this is was a perfect use case to utilize AI to reformat the existing data.
Used Gemini (LLM) and generated a prompt to reformat variable names, codes, and labels into a structured table format that could be exported into a CSV format (code_mappings.csv) for use as a lookup table.


### Database Setup in SSMS

Loaded the survey data into SQL Server Management Studio (SSMS) using the management interface.
Created [dbo].[Responses] table, altering default TINYINT columns to INT to handle high codes.

Added a primary key on SEQID

```
ALTER TABLE [CPADS].[dbo].[Responses]
ADD PRIMARY KEY (SEQID);
```

Created [dbo].[Codebook] table with Variable_Name (NVARCHAR), Code (INT), and Label (NVARCHAR)

```
CREATE TABLE Codebook (
    [Variable_Name] nvarchar(50),
    [Code] int,
    [Label] nvarchar(MAX)
);
```

Imported codebook data using BULK INSERT

```
BULK INSERT [CPADS].[dbo].[Codebook]
FROM 'C:\Users\tabor\code_mappings.csv'
WITH (FIRSTROW = 2, FIELDTERMINATOR = ',', ROWTERMINATOR = '\n');
```

### Variable Selection

Selected 15 key variables for analysis, focusing on demographics, health, and substance use:

1. SEQID: Unique case identifier.
2. stu04: Enrollment status (Full-time/Part-time).
3. region: School region.
4. age_groups: Age categories.
5. dvdemq01: Gender.
6. dvdemq6: Ethnicity.
7. hwbq01: Physical health rating.
8. hwbq02: Mental health rating.
9. alc05: Alcohol use (past 12 months).
10. alc06: Alcohol frequency (past 30 days).
11. alc10: Alcohol drinks per occasion (past 30 days).
12. can05: Cannabis use (past 12 months).
13. can06: Cannabis frequency (past 30 days).
14. dru01_f: Hallucinogen use (ever).
15. vap01: Vaping use (ever).


### SQL View Creation:

To join up the labels with the raw data, created [dbo].[CPADS_Decoded] view to join [dbo].[Responses] with [dbo].[Codebook], decoding codes into readable labels (e.g., stu04 → Enrollment_Status_Code)

```
CREATE OR ALTER VIEW [dbo].[CPADS_Decoded] AS
SELECT 
    r.SEQID AS Case_ID,
    r.stu04 AS Enrollment_Status_Code,
    s.Label AS Enrollment_Status_Label,
    r.region AS School_Region_Code,
    reg.Label AS School_Region_Label,
    r.age_groups AS Age_Category_Code,
    a.Label AS Age_Category_Label,
    r.dvdemq01 AS Gender_Code,
    g.Label AS Gender_Label,
    r.dvdemq6 AS Ethnicity_Code,
    e.Label AS Ethnicity_Label,
    r.hwbq01 AS Physical_Health_Rating_Code,
    ph.Label AS Physical_Health_Rating_Label,
    r.hwbq02 AS Mental_Health_Rating_Code,
    mh.Label AS Mental_Health_Rating_Label,
    r.alc05 AS Alcohol_Past_12M_Code,
    alc5.Label AS Alcohol_Past_12M_Label,
    r.alc06 AS Alcohol_30D_Frequency_Code,
    alc6.Label AS Alcohol_30D_Frequency_Label,
    r.alc10 AS Alcohol_30D_Drinks_Code,
    alc10.Label AS Alcohol_30D_Drinks_Label,
    r.can05 AS Cannabis_Past_12M_Code,
    can5.Label AS Cannabis_Past_12M_Label,
    r.can06 AS Cannabis_30D_Frequency_Code,
    can6.Label AS Cannabis_30D_Frequency_Label,
    r.dru01_f AS Hallucinogen_Ever_Used_Code,
    hal.Label AS Hallucinogen_Ever_Used_Label,
    r.vap01 AS Vaping_Ever_Used_Code,
    vap.Label AS Vaping_Ever_Used_Label
FROM [dbo].[Responses] r
LEFT JOIN [dbo].[Codebook] s ON s.Variable_Name = 'stu04' AND r.stu04 = s.Code
LEFT JOIN [dbo].[Codebook] reg ON reg.Variable_Name = 'region' AND r.region = reg.Code
LEFT JOIN [dbo].[Codebook] a ON a.Variable_Name = 'age_groups' AND r.age_groups = a.Code
LEFT JOIN [dbo].[Codebook] g ON g.Variable_Name = 'dvdemq01' AND r.dvdemq01 = g.Code
LEFT JOIN [dbo].[Codebook] e ON e.Variable_Name = 'dvdemq6' AND r.dvdemq6 = e.Code
LEFT JOIN [dbo].[Codebook] ph ON ph.Variable_Name = 'hwbq01' AND r.hwbq01 = ph.Code
LEFT JOIN [dbo].[Codebook] mh ON mh.Variable_Name = 'hwbq02' AND r.hwbq02 = mh.Code
LEFT JOIN [dbo].[Codebook] alc5 ON alc5.Variable_Name = 'alc05' AND r.alc05 = alc5.Code
LEFT JOIN [dbo].[Codebook] alc6 ON alc6.Variable_Name = 'alc06' AND r.alc06 = alc6.Code
LEFT JOIN [dbo].[Codebook] alc10 ON alc10.Variable_Name = 'alc10' AND r.alc10 = alc10.Code
LEFT JOIN [dbo].[Codebook] can5 ON can5.Variable_Name = 'can05' AND r.can05 = can5.Code
LEFT JOIN [dbo].[Codebook] can6 ON can6.Variable_Name = 'can06' AND r.can06 = can6.Code
LEFT JOIN [dbo].[Codebook] hal ON hal.Variable_Name = 'dru01_f' AND r.dru01_f = hal.Code
LEFT JOIN [dbo].[Codebook] vap ON vap.Variable_Name = 'vap01' AND r.vap01 = vap.Code;
```

## Power BI Integration:

Connected to SQL database in Power BI using Import mode.
In Power Query Editor (PQE), filtered Enrollment_Status_Label to include only "Full-Time" and "Part-Time" students.
Performed data cleaning in PQE to remove unnecessary characters (e.g., quotations, commas) from label columns.
Since these transformations are all native to SQL the query can be folded back to the server and will not be run inside Power BI, increasing efficiency.


## Visualizations

Created exploratory visualizations in Power BI to analyze substance use patterns:
Bar Chart: Prevalence of alcohol, cannabis, hallucinogen, and vaping use by gender, using measures:

```
Pct_Alcohol_Past_12M = 
DIVIDE(
    CALCULATE(COUNT('CPADS_Decoded'[Case_ID]), 'CPADS_Decoded'[Alcohol_Past_12M_Label] = "Yes"),
    COUNT('CPADS_Decoded'[Case_ID]),
    0
) * 100
```


Pie Chart: Vaping prevalence by age category, highlighting trends in younger students.
Heatmap (Table): Substance use by ethnicity and physical health, using conditional formatting to highlight high-use groups.




