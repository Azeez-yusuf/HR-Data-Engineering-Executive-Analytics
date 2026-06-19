# HR-Data-Engineering-Executive-Analytics
This project is undertaken solely for academic and exploratory purposes. Its objective is to analyze the HR dataset to gain valuable insights into employee retention patterns and to identify the various factors that influence workforce retention within the organization.

## The Raw Data
The dataset used for this project was obtained from Kaggle. Although the original dataset consisted of a single CSV file, it was divided into two separate files to better support the academic objectives and analytical requirements of the study.

To ensure clarity, consistency, and ease of analysis, the raw data underwent a preprocessing phase. This process involved redefining certain variables, removing irrelevant columns, and creating additional fields where necessary to enhance the quality and relevance of the analysis.


## Tools and Steps
•	EXCEL; Used for:
1. Data loading and inspection
2. Data manipulations
3. Handling missing values,
4. Cleaning and
5. Formatting

•  SQL; Steps took involved:
1. Loading the refined datasets into the Database
2. Feature Engineering
3. Analysis

• Power BI; Steps took involved:
1. Transformation
2. Feature Engineering
3. Analysis
4. Visualizations

## Challenges
•	Challenge  1: Several improper casing of alphabets, a lot of misspellings, duplicates columns and rows, values errors, nulls and missing values were observed in the raw dataset.
      o	Solution: I use MS EXCEL to perform data manipulations, transformations, cleaning and formatting

•	Challenge 2: Another hurdle was loading the dataset to SQL server base, this was as a result of incorrect formatting of the CSV file to correspond with the SQL data type format
      o	Solution: This was also handled using Excel and SQL

• Challenge 3: After successive analysis in Sql, Creating successful relationships between exported files was herculian task in Power BI
      o	Solution: This was done by noting the type of relationship between the export files and applying them
      
## Feature Engineering
Feature engineering in SQL for data analysis involves strategically crafting new, informative features from existing data to enhance model performance and extract deeper insights.

## Feature Engineering CODES

#### DataBase Creation AND Importation Process

```sql

CREATE DATABASE IF NOT EXISTS delofworkers;
USE delofworkers;

-- Create the Dimension Table for Demographics
DROP TABLE employee_demographics;
CREATE TABLE IF NOT EXISTS employee_demographics (
							EmpID INT NOT NULL,
							Employee_Name VARCHAR(100) NOT NULL,
							DOB DATE NOT NULL,
							Sex VARCHAR(10) NOT NULL,
							MaritalDesc VARCHAR(30) NOT NULL,
							CitizenDesc VARCHAR(50) NOT NULL,
							HispanicLatino VARCHAR(10) NOT NULL,
							RaceDescript VARCHAR(50) NOT NULL,
							State VARCHAR(10) NOT NULL,
							PRIMARY KEY (EmpID)
							) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
                            

-- Create the Operational/Fact Table for Employment Details
DROP TABLE employment_details;
CREATE TABLE IF NOT EXISTS employment_details (
							SerialNumber INT NOT NULL,
                            EmpID INT NOT NULL,
							Salary INT NOT NULL,
							PositionID INT NOT NULL,
							Position VARCHAR(100) NOT NULL,
							Department VARCHAR(50) NOT NULL,
							ManagerName VARCHAR(100) NOT NULL,
							ManagerID INT NULL,                    -- Uses NULL for records without a manager
							DateofHire DATE NOT NULL,
							DateofTermination DATE NULL,           -- Uses NULL for active employees
							TermReason VARCHAR(255) NOT NULL,
							EmploymentStatus VARCHAR(50) NOT NULL,
							RecruitmentSource VARCHAR(50) NOT NULL,
							PerformanceScore VARCHAR(50) NOT NULL,
							EngagementSurvey DECIMAL(3,2) NOT NULL, -- Perfect for handling values like 4.60 or 3.02
							EmpSatisfaction INT NOT NULL,
							SpecialProjectsCount INT NOT NULL,
							LastPerformanceReview_Date DATE NOT NULL,
							DaysLateLast30 INT NOT NULL,
							Absences INT NOT NULL,
							PRIMARY KEY (SerialNumber),
							FOREIGN KEY (EmpID) REFERENCES employee_demographics(EmpID) 
								ON DELETE CASCADE 
								ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

### FEATURE ENGINEERING
-- --------------------------------------------------------------------------------------
-- --------------------------------------------------------------------------------------
-- ----------------FEATURE ENGINEERING-----------------------
-- ------------Post-Ingestion Data Standardization Script------------------------------------------


```sql

-- Turn off the safety guardrail ---------------------------------
SET SQL_SAFE_UPDATES = 0;

-- 1. Clean Demographics Table------------------------------------------------------------------------------------

UPDATE employee_demographics
SET
	Sex = TRIM(Sex),
    Employee_Name = TRIM(Employee_Name);
    
    
-- 2. Clean Employment Details Table --------------------------------------------------------------------------------
UPDATE employment_details
SET
	Department = TRIM(Department),
    -- Convert text 'NULL' to native database NULL
    ManagerID = CASE WHEN TRIM(ManagerID) = 'NULL' THEN NULL ELSE TRIM(ManagerID) END,
    DateofTermination = CASE WHEN TRIM(DateofTermination) = 'NULL' THEN NULL ELSE TRIM(DateofTermination) END;
    
    
-- 3. Alter Column Data Types to Optimal Formats now that 'NULL' strings are gone ------------------------------------- 
ALTER TABLE employment_details 
    MODIFY COLUMN ManagerID INT NULL,
    MODIFY COLUMN DateofTermination DATE NULL;
    
 
 
 --  Turn on the safety guardrail ---------------------------------------------
SET SQL_SAFE_UPDATES = 1;

```


-- OVERVIEW OF THE WHOLE TABLES IN THE DATABASE----------------------------------------------------------------------------

```sql

SELECT *
FROM employee_demographics;

SELECT *
FROM employment_details;

```


-- ----------------------------------------------------------------------------------------------------------------------------

-- CORE ANALYSIS ---------------------------------------------------------------------------------------------------------------


```sql

-- The Executive HR Dashboard (High-Level KPIs): a birds-eye view of the organization's health
SELECT
    COUNT(*) AS Total_Historical_Headcount,              																		 -- This is the total number of people who have ever worked here (both past and present employees)
    SUM(CASE WHEN EmploymentStatus = 'Active' THEN 1 ELSE 0 END) AS Active_Employees,    										 -- This is your current team. The number of people who currently work here right now
    SUM(CASE WHEN EmploymentStatus <> 'Active' THEN 1 ELSE 0 END) AS Terminated_Employees,  									 -- This is the total number of people who have left the company over time
    ROUND((SUM(CASE WHEN EmploymentStatus <> 'Active' THEN 1 ELSE 0 END) / COUNT(*)) * 100, 2) AS Cumulative_Turnover_Rate_Pct,  -- This tells you what percentage of your total workforce has left the company since day one. A high percentage means people leave quickly; a low percentage means people stay
    ROUND(AVG(Salary), 2) AS Average_Salary,    																				 -- The average amount of money a worker makes here
    ROUND(AVG(EmpSatisfaction), 2) AS Average_Employee_Satisfaction, 															 -- On average, how happy are (or were) the employees working here?
    ROUND(AVG(EngagementSurvey), 2) AS Average_Engagement_Score,    															 -- On average, how motivated and checked-in are the employees?
    SUM(SpecialProjectsCount) AS Total_Active_Special_Projects      															 -- The total number of extra, special projects the company has successfully assigned across the workforce.
FROM employment_details;




-- -----------------------------------------------------------------------------------------------------------------------------------------------------------
-- Workforce Attrition & Tenure Friction Points: Retaining talent is significantly cheaper than sourcing it. Since this is a historical dataset where the latest recorded date is in early 2019, active tenure is calculated using '2019-03-01' as the baseline study date
SELECT 
    ed.Department,
    COUNT(CASE WHEN ed.EmploymentStatus <> 'Active' THEN 1 END) AS Total_Terminations,                                      -- This counts how many people quit or were let go from this specific department.
    ROUND(AVG(CASE 
        WHEN ed.EmploymentStatus <> 'Active' THEN TIMESTAMPDIFF(MONTH, ed.DateofHire, ed.DateofTermination)                 -- TIMESTAMPDIFF(MONTH, Date A, Date B): is just a stopwatch that calculates exactly how many months passed between Date A and Date B. This is used to calculate Tenure (how long someone stayed at the company).
    END), 1) AS Avg_Tenure_Months_Left_Employees,                                                                           -- Avg_Tenure_Months_Left_Employees: For the people who quit this department, how long did they usually last before walking out the door?" (e.g., If the answer is 6.5, they usually quit after just half a year).
    ROUND(AVG(CASE 
        WHEN ed.EmploymentStatus = 'Active' THEN TIMESTAMPDIFF(MONTH, ed.DateofHire, '2019-03-01')
    END), 1) AS Avg_Tenure_Months_Active_Employees,                                                                         -- Avg_Tenure_Months_Active_Employees:  For the people currently working in this department right now, how long have they been with us so far?
    -- Identifies the most common reason for leaving within that department
    (SELECT TermReason 
     FROM employment_details sub_ed 
     WHERE sub_ed.Department = ed.Department AND sub_ed.EmploymentStatus <> 'Active'
     GROUP BY TermReason 
     ORDER BY COUNT(*) DESC 
     LIMIT 1) AS Primary_Reason_For_Leaving
FROM employment_details ed
GROUP BY ed.Department                                                                                          -- The first place any calculation happens. Everything calculated below happens inside those specific piles
ORDER BY Total_Terminations DESC;


-- -------------- -----------------------------------------------------------------------------------------------------------------------------------------------------
-- Recruitment Channel Performance & Talent ROI
/*
 Not all recruiting sources are created equal. This query grades your pipelines (RecruitmentSource) by 
evaluating the cost of talent (Salary) versus output metrics like performance and retention.To calculate an accurate average performance score,
we convert the text-based rating into an ordinal numeric index where $\text{PIP} = 1$, $\text{Needs Improvement} = 2$, $\text{Fully Meets} = 3$, and $\text{Exceeds} = 4$.
*/


SELECT 
    RecruitmentSource,
    COUNT(*) AS Total_Hires,
    ROUND(AVG(Salary), 2) AS Average_Salary,
    ROUND(AVG(CASE 
        WHEN PerformanceScore = 'Exceeds' THEN 4
        WHEN PerformanceScore = 'Fully Meets' THEN 3
        WHEN PerformanceScore = 'Needs Improvement' THEN 2
        WHEN PerformanceScore = 'PIP' THEN 1
        ELSE NULL 
    END), 2) AS Avg_Performance_Index,
    ROUND(AVG(EngagementSurvey), 2) AS Avg_Engagement_Score,
    ROUND(AVG(Absences), 1) AS Avg_Absences_Per_Employee,
    ROUND((SUM(CASE WHEN EmploymentStatus = 'Voluntarily Terminated' THEN 1 ELSE 0 END) / COUNT(*)) * 100, 2) AS Voluntary_Attrition_Rate_Pct
FROM employment_details
GROUP BY RecruitmentSource                                                                                                                          -- As usual; The Sorting
HAVING Total_Hires >= 5 -- Filters out channels with low sample size for statistical validity
ORDER BY Avg_Performance_Index DESC;




-- ------------------------------------------------------------------------------------------------------------------------------------------------
-- Leadership Scorecard & Operational Risk Index-----------------------------------
/* Employees don't quit companies; they quit managers. This query serves as a supervisory health audit, 
identifying which managers lead highly engaged, reliable teams and which ones may be running teams at high risk of burnout or absenteeism.
*/

SELECT 
    ManagerName,
    COUNT(*) AS Direct_Reports,
    ROUND(AVG(EmpSatisfaction), 2) AS Team_Satisfaction_Avg,
    ROUND(AVG(EngagementSurvey), 2) AS Team_Engagement_Avg,
    ROUND(AVG(DaysLateLast30), 2) AS Avg_Days_Late_Last_30,
    ROUND(AVG(Absences), 2) AS Avg_Absences,
    SUM(CASE WHEN PerformanceScore IN ('Needs Improvement', 'PIP') THEN 1 ELSE 0 END) AS Underperforming_Count
FROM employment_details
GROUP BY ManagerName
HAVING Direct_Reports > 2
ORDER BY Team_Satisfaction_Avg ASC;                            -- I am us Ascending order to Displays lowest satisfaction scores first to highlight urgent risks




-- ----------------------------------------------------------------------------------------------------------------------------------------------------
-- Compensation Equity & Diversity Audit
/* Modern enterprise analytics requires continuous auditing of systemic compensation equity. This query cross-references demographic identities against
 compensation bands within the same departments to provide visibility into structural fairness.
*/


SELECT 
    emp.Department,
    demo.Sex,
    demo.RaceDescript,
    COUNT(*) AS Headcount,
    ROUND(AVG(emp.Salary), 2) AS Average_Salary,
    ROUND(MIN(emp.Salary), 2) AS Min_Salary,
    ROUND(MAX(emp.Salary), 2) AS Max_Salary,
    ROUND(MAX(emp.Salary) - MIN(emp.Salary), 2) AS Pay_Spread
FROM employment_details emp
JOIN employee_demographics demo ON emp.EmpID = demo.EmpID
GROUP BY emp.Department, demo.Sex, demo.RaceDescript
ORDER BY emp.Department, Average_Salary DESC;


```



-- ----------------------------------------------------------------------------------------------------------------------
## Results/Findings
##### Data Insight Generation for The Analysis

1. The dataset reflects a historical workforce headcount of 311 employees, comprising 207 active employees and 104 terminated employees. This represents a cumulative turnover rate of 33.44%, indicating that approximately one-third of the workforce has exited the organization over the period covered by the dataset.

Analysis of employee compensation reveals an average salary of $69,020.68 across the workforce. Employee satisfaction levels are relatively strong, with an average satisfaction score of 3.89 out of 5.00. Similarly, employee engagement is high, with an average engagement score of 4.11 out of 5.00, suggesting a generally positive employee experience.

Additionally, the dataset records a total of 379 active special projects, highlighting a significant level of employee involvement in initiatives beyond routine operational responsibilities.


2. The workforce attrition story is not primarily about employees being fired across all departments. Instead, it appears to be driven by retention challenges in Production, where employees frequently leave for other opportunities, while IT-IS and Admin Offices face early-tenure employee quality and performance issues. Focusing retention efforts on Production and improving hiring, onboarding, and performance management in IT-IS and Admin Offices would likely yield the greatest reduction in turnover.


3. The strongest recruitment ROI comes from Employee Referrals, LinkedIn, and Indeed, which consistently produce employees with higher performance and lower voluntary attrition. Conversely, Diversity Job Fairs, Google Search, and CareerBuilder generate significantly higher turnover rates, indicating potential issues with candidate-job fit, onboarding effectiveness, or recruitment targeting. Expanding referral programs while reviewing the quality and retention outcomes of high-attrition recruitment channels would likely improve overall workforce stability and reduce hiring costs.


4. The leadership data suggests overall workforce engagement remains healthy across most teams, but employee satisfaction varies significantly by manager. The highest operational risk sits with Brannon Miller, whose large team records the lowest satisfaction score and the highest concentration of underperforming employees. Attendance risk is most evident under John Smith, whose team reports the highest absenteeism levels. Conversely, Ketsia Liebig, Alex Sweetwater, and Jennifer Zamora demonstrate the strongest leadership outcomes, combining high satisfaction, solid engagement, and low underperformance. The broader organizational pattern indicates employees remain productive and engaged, but declining satisfaction in selected teams could become a future attrition driver if leadership effectiveness and employee experience are not addressed.


   ## STRATEGIC FINDINGS AND RECOMMENDATIONS

-- ----------------------------------------------------------------------------------------------------------------------------------
-- ----------------------------------------------------------------------------------------------------------------------------------
A. 

/*
-- The "Flight Risk" Red Line: This is to Identify the peak tenure window (in months) where employees voluntarily resign, 
and evaluate their average salary and performance footprint during that critical window.

-- Query 1: Voluntary Attrition Tenure Distribution & Compensation Footprint
*/

```sql

SELECT 
    CASE 
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 12 THEN '0-1 Year'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 24 THEN '1-2 Years'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 36 THEN '2-3 Years'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 48 THEN '3-4 Years'
        ELSE '4+ Years'
    END AS Tenure_Group_At_Departure,
    COUNT(*) AS Total_Voluntary_Resignations,
    ROUND(AVG(Salary), 2) AS Avg_Salary_At_Departure,
    ROUND(AVG(EngagementSurvey), 2) AS Avg_Engagement_Score,
    ROUND(AVG(CASE 
        WHEN PerformanceScore = 'Exceeds' THEN 4
        WHEN PerformanceScore = 'Fully Meets' THEN 3
        WHEN PerformanceScore = 'Needs Improvement' THEN 2
        WHEN PerformanceScore = 'PIP' THEN 1
    END), 2) AS Avg_Performance_Index
FROM employment_details
WHERE EmploymentStatus = 'Voluntarily Terminated'
GROUP BY 
    CASE 
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 12 THEN '0-1 Year'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 24 THEN '1-2 Years'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 36 THEN '2-3 Years'
        WHEN TIMESTAMPDIFF(MONTH, DateofHire, DateofTermination) <= 48 THEN '3-4 Years'
        ELSE '4+ Years'
    END
ORDER BY Total_Voluntary_Resignations DESC;

```


##### 🚨Crisis 1: The 4+ Years "Loyalty Drain" (37.5% of all exits)
The Data: $33$ out of $88$ total resignations come from employees who have been with the company for over 4 years. They have healthy engagement ($4.19$) and reliable performance ($2.91$). However, they have the lowest average salary at departure ($\$57,671.52$).
The Interpretation for Stakeholders: These are our loyal, long-term corporate citizens. They perform well and are engaged, but they are leaving because they are severely underpaid. This is a classic textbook case of Salary Compression—where market wages for new hires rise quickly, but internal raises for existing employees stagnate. They are eventually forced to leave to get the market rate they deserve.
##### 🚨 Crisis 2: The 3-4 Years "Star Performer Exit" (19.3% of all exits)
The Data: $17$ employees resigned during their 3rd to 4th year. This group represents our absolute rockstars: they have the highest performance index ($3.18$), the highest engagement score ($4.40$), and the highest average salary ($\$71,066.06$).
The Interpretation for Stakeholders: Money is not the primary reason these individuals are leaving; we are paying them premium rates. Because they are highly engaged and high-performing, they are highly marketable. They are likely being headhunted by competitors. They are leaving because they have hit a "career ceiling" with us and are exiting to find higher-level leadership or growth opportunities that we aren't providing internally.
##### ⚠️ The 1-2 Years "Reality Check" Dip (18.2% of all exits)
The Data: After a stable first year (only $9$ resignations), departures nearly double to $16$ in the 1–2 year window. During this phase, engagement drops to $4.08$ and performance dips to its lowest point ($2.88$).
The Interpretation for Stakeholders: This represents an early-stage friction point. After the "honeymoon phase" of the first year wears off, employees experience a dip in enthusiasm and performance. This usually points to a lack of structured mid-tenure support or a mismatch between what was promised during recruitment and the actual day-to-day job reality.


#### 3 clear action items to management based on this data:
1. Execute a "Loyalty Equity" Pay Adjustment: Immediately audit the compensation of all current employees with over 3 years of tenure. Bringing underpaid veterans up to current market rates will instantly neutralize the largest exit group ($37.5\%$).
2. Introduce Fast-Track Growth Paths for High Performers: Implement a formal talent review for employees hitting their 2-to-3 year marks. Ensure our top performers (the 3-4 year group) are given clear internal promotional tracks, executive mentorship, or new challenges so they don’t look elsewhere for growth.
3. Launch a Year-2 "Re-boarding" Program: Address the 1-2 year retention leak by introducing a milestone check-in at the 12-month mark. Use this to realign expectations, address disengagement, and provide fresh training to rebuild their performance momentum.
-- ---------------------------------------------------------------------------------------------------------------------------------------


-- ------------------------------------------------------------------------------------------------------------------------------------
B. 

/*
The Absenteeism & Burnout Correlation: Objective: Map out whether a high frequency of absences and lateness correlates directly with
 low satisfaction scores, and isolate which departments or managers are driving these burnout metrics.
 
 */
 
 -- Query 2: Absenteeism, Lateness, and Dissatisfaction Hotspots

 ``sql
 
SELECT 
    Department,
    ManagerName,
    COUNT(*) AS Team_Size,
    ROUND(AVG(Absences), 1) AS Avg_Absences_Per_Employee,
    ROUND(AVG(DaysLateLast30), 1) AS Avg_Days_Late_Last_30,
    ROUND(AVG(EmpSatisfaction), 2) AS Avg_Employee_Satisfaction,
    -- Composite Risk Flag: High absences/lateness paired with satisfaction below 3.5
    SUM(CASE WHEN (Absences > 10 OR DaysLateLast30 > 2) AND EmpSatisfaction < 4 THEN 1 ELSE 0 END) AS High_Burnout_Risk_Count
FROM employment_details
GROUP BY Department, ManagerName
HAVING Team_Size >= 3
ORDER BY High_Burnout_Risk_Count DESC, Avg_Employee_Satisfaction ASC;

```

#### Executive Summary: The Burnout & Attendance Bottlenecks
A high-level view reveals that burnout isn’t randomly scattered—it is highly concentrated. In fact, Production and IT-IS account for 88% of all high-burnout cases in the company (58 out of 66 total employees at risk).
##### 🚨 The "Red Zone" Teams (Burnout Rate > 30%)
When we look at the percentage of a team at risk, three managers require immediate operational intervention:
1. Brannon Miller (Production): Exactly 50% of this team is at high risk of burnout ($11$ out of $22$ employees). This team also exhibits a high friction rate with an average of 1.0 day late over the last 30 days. Their satisfaction score is the lowest in Production ($3.41$).
2. Janet King (Production): 40% of the team is at high risk ($6$ out of $15$ employees). They also suffer from low satisfaction ($3.47$).
3. Brian Champaigne (IT-IS): 37.5% of this team is at high risk ($3$ out of $8$ employees). Alarmingly, this team has an incredibly high absenteeism rate, averaging 12.5 absences per employee.

#### 🔍 Key Insight: The Lateness vs. Absence Behavioral Signal
The data reveals a fascinating trend in how burnout manifests across different job functions:
1. The "Check-Out" Pattern (Sales & IT): In Sales and IT-IS, burned-out employees simply stop showing up as often. John Smith’s Sales team averages a massive 13.4 absences per employee. This indicates that in desk-bound or client-facing roles, burnout correlates with checking out completely (absences).
2. The "Tardy" Pattern (Production): In Production, where shifts are rigid, employees try to show up, but they arrive late. Brannon Miller's team leads the company in lateness. This indicates operational fatigue—people are exhausted, making it physically harder to start the shift on time.


####💡 Strategic Recommendations for Leadership
1. Investigate Leadership & Workloads under Brannon Miller & Janet King: Production has many managers with healthy teams (e.g., Ketsia Liebig and Michael Albert have under 10% burnout rates). Because some Production teams are thriving while Miller's and King's are struggling, this is likely a localized shift-scheduling, workload distribution, or management style issue rather than a company-wide problem.
2. Deploy "Early Warning" HR Interventions based on Lateness/Absences: HR should build an automated alert. If a team’s average monthly lateness crosses $0.5$ days, or individual absences cross $11$ days, it should trigger an automatic wellness and workload audit. The data proves these are the behavioral footprints of a burning-out team.
3. Cross-Train High-Absence Teams: Teams like John Smith's (Sales) and Brian Champaigne's (IT) are running thin due to double-digit employee absences. Leadership must ensure there is adequate coverage so that one employee's absence doesn't cause a domino effect of burnout on the remaining team members.

