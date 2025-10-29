# Student-Performance-Analysis

## Table of Contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Data Cleaning/Preparation](#data-cleaningpreparation)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Final Results](#final-results)

## Project Overview  
This project involves the cleaning, preparation, and exploratory data analysis (EDA) of a multi-table student dataset sourced from a MySQL database. The primary goal was to practice and showcase an end-to-end data analysis workflow, including database interaction, handling data inconsistencies, performing EDA using SQL and Python, and deriving basic metrics.

Disclaimer: Upon analysis, the dataset exhibits significant uniformity across various metrics (e.g., attendance rates, homework completion, exam scores across subjects/grades), suggesting it is likely synthetic or heavily anonymized/modified. Therefore, the emphasis of this project is on the data processing and analysis methodology rather than deriving real-world behavioral insights about student performance.

## Data Sources

The data were taken from [Kaggle](https://www.kaggle.com/datasets/marvyaymanhalim/student-performance-and-attendance-dataset/) and stored in a MySQL database containing five tables:

- attendance: Records of student attendance status per subject and date.

- homework: Details on homework assignments, due dates, status, grades, and guardian signatures.

- performance: Exam scores and homework completion percentages per student and subject.

- communication: Logs of messages between parents and teachers.

- students: Basic student demographic information like name, date of birth, and grade level.

## Tools 
- **MySQL**
- **Python** 
- **pandas**: Data manipulation and analysis.
- **numpy**: Numerical operations.
- **matplotlib & seaborn**: Data visualization (used minimally for correlation check).
- **sqlalchemy**: Connecting to and interacting with the MySQL database.
- **scipy.stats**: For statistical calculations (Pearson correlation).

## Data Cleaning/Preparation  

Significant cleaning was required across all tables to ensure consistency and usability:

* **Date Formatting:** Standardized mixed date formats (e.g., `YYYY-MM-DD`, `MM/DD/YYYY`, `DD/MM/YYYY`) into a consistent datetime object using `pandas.to_datetime` with `format='mixed'`.
* **Categorical String Cleaning:**
    * Trimmed leading/trailing whitespace using `.str.strip()`.
    * Standardized inconsistent values in `attendance_status` (e.g., 'PRESENT ', 'Present', 'present' unified) and `homework.status` (e.g., '✅', '✔', 'Done' unified) using custom mapping functions.
* **Numerical Cleaning:**
    * Corrected invalid `exam_score` entries (values > 100 were removed).
    * Standardized `homework_completion` by removing '%' signs and converting to integers; removed nonsensical negative values.
    * Converted letter grades (`homework.grade_feedback`) to numerical GPA values using a mapping dictionary.
* **Missing/Placeholder Values:** Replaced empty strings or spaces (e.g., in `guardian_signature`) with `NaN`. Removed records with essential empty fields (e.g., `communication.message_content`).
* **Data Type Conversion:** Converted relevant columns to appropriate types (e.g., `students.grade_level` from 'Grade X' string to integer).
* **Irrelevant Column Removal:** Dropped columns like `teacher_comments`, `message_content`, and `emergency_contact` that were not used in the subsequent analysis.
* **Database Update:** Loaded the cleaned dataframes back into new tables in the MySQL database (e.g., `attendance_cleaned`).  

## Exploratory Data Analysis

EDA was performed using both direct SQL queries (via `pd.read_sql`) and pandas operations to understand the data characteristics:

* **Basic Metrics:** Checked overall date ranges, total student counts, and student distribution per grade level.
* **Performance Exploration:** Calculated average exam scores and homework completion rates per subject. Identified students with the highest (100) and lowest (40) average scores.
* **Attendance Patterns:** Analyzed attendance status distribution per subject, identifying overall rates (Attended, Absent, Excused). Checked for patterns across students, grade levels, and seasons.
* **Homework & Engagement:** Investigated homework completion status (Done, Not Done, Pending) overall and per subject. Analyzed average GPA per subject. Examined guardian signature rates overall, by grade level, and in relation to homework status.
* **Correlation Checks:** Used `scipy.stats.pearsonr` and `seaborn.scatterplot` to check for correlations between:
    * Homework completion and exam scores.
    * Attendance rates and exam scores.

**Key Observation:** A notable finding during EDA was the extreme uniformity in distributions and averages across different subjects, grades, and even time periods (seasons). Correlations between expected factors (like attendance/homework completion and scores) were negligible (close to zero correlation coefficient and high p-values). This uniformity strongly indicates the synthetic nature of the data.

## Data Analysis

### At-Risk Student Identification
As an example of synthesizing the data, basic criteria were defined to identify potentially at-risk students based on combined metrics:

* **Thresholds:**
    * Attendance Rate < 75%
    * Average Exam Score < 60
    * Homework Completion Rate < 70%
* **Risk Levels:**
    * **High Risk:** Meets all three criteria.
    * **Moderate Risk:** Meets the low score criterion plus one other criterion.
    * **Low Risk:** Meets one or none of the criteria.
``` python
query = """
WITH 
-- Attendance rate per student
attendance_summary AS (
    SELECT 
        student_id,
        ROUND(
          SUM(CASE WHEN attendance_status IN ('Late','Left Early','Present') THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
        , 2) AS attendance_rate
    FROM attendance_cleaned
    GROUP BY student_id
),

-- Exam score average per student
exam_summary AS (
    SELECT 
        student_id,
        ROUND(AVG(exam_score), 2) AS average_exam_score
    FROM performance_cleaned
    GROUP BY student_id
),

-- Homework completion and guardian signature rates per student
homework_summary AS (
    SELECT 
        student_id,
        ROUND(SUM(CASE WHEN status = 'Done' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS homework_completion_rate,
        ROUND(AVG(guardian_signature) * 100, 2) AS guardian_signature_rate
    FROM homework_cleaned
    GROUP BY student_id
),

-- Combine all metrics into one summary table
student_summary AS (
    SELECT 
        s.student_id,
        s.full_name,
        a.attendance_rate,
        e.average_exam_score,
        h.homework_completion_rate,
        h.guardian_signature_rate
    FROM students_cleaned s
    LEFT JOIN attendance_summary a ON s.student_id = a.student_id
    LEFT JOIN exam_summary e ON s.student_id = e.student_id
    LEFT JOIN homework_summary h ON s.student_id = h.student_id
)

-- Classify risk levels
SELECT
    student_id,
    full_name,
    attendance_rate,
    average_exam_score,
    homework_completion_rate,
    guardian_signature_rate,
    CASE
        WHEN attendance_rate < 75 
          AND average_exam_score < 60 
          AND homework_completion_rate < 70 THEN 'High Risk'
        WHEN (attendance_rate < 75 AND average_exam_score < 60)
          OR (average_exam_score < 60 AND homework_completion_rate < 70)
          THEN 'Moderate Risk'
        ELSE 'Low Risk'
    END AS risk_level
FROM student_summary
WHERE average_exam_score IS NOT NULL  -- keep only students with exam data
ORDER BY 
    CASE risk_level
        WHEN 'High Risk' THEN 1
        WHEN 'Moderate Risk' THEN 2
        ELSE 3
    END,
    average_exam_score ASC;
"""

df_summary = pd.read_sql(query, engine)
df_summary.sample(5)
```

A final summary table (`df_summary`) was created by joining the cleaned student information with calculated attendance rates, average exam scores, and homework completion/signature rates, and the risk level was assigned using a SQL `CASE` statement.

## Final Results

* **Cleaned Data:** Five new tables (`_cleaned` suffix) were created in the MySQL database containing the processed and standardized data.
* **Summary Table:** A summary dataframe (`df_summary`) was generated, aggregating key performance and engagement metrics for each student and assigning a calculated risk level.
* **No Substantive Findings:** Due to the apparent synthetic nature of the data, no significant correlations or meaningful variations in student performance/behavior were identified. The "results" primarily demonstrate the successful application of the data processing and analysis steps.
