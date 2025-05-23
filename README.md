# Data Analysis using World Layoffs SQL Project

## Project Overview
This project uses the Layoffs 2022 dataset from Kaggle to analyze global tech layoffs using MySQL. The goal is to clean, transform, and perform exploratory data analysis (EDA) to extract meaningful insights from layoff trends across different companies, industries, and geographies.

*Dataset*: https://www.kaggle.com/datasets/swaptr/layoffs-2022

### 1) Database Setup

```sql
CREATE DATABASE world_layoffs;
USE world_layoffs;

-- Import the dataset into a table named 'layoffs'

SELECT * FROM layoffs;
```

### 2) Data Cleaning & Transformation
Step 1: Create Staging Table
```sql
CREATE TABLE layoffs_staging LIKE layoffs;

INSERT layoffs_staging
SELECT * FROM layoffs;
```

Step 2: Remove Duplicates
Since MySQL doesn't allow deleting directly from a CTE, we use a secondary staging table:
```sql
CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `total_laid_off` double DEFAULT NULL,
  `date` text,
  `percentage_laid_off` text,
  `industry` text,
  `source` text,
  `stage` text,
  `funds_raised` text,
  `country` text,
  `date_added` text,
  `row_num` INT
);

-- Insert with row numbers for duplicate identification
INSERT INTO layoffs_staging2
SELECT *, ROW_NUMBER() OVER
( PARTITION BY company, location, total_laid_off, `date`, percentage_laid_off, industry, `source`, stage, funds_raised, country, date_added) AS row_num
FROM layoffs_staging;

SELECT * FROM layoffs_staging2;

-- Remove duplicates
DELETE FROM layoffs_staging2
WHERE row_num > 1;
```

Step 3: Standardize & Clean Data
Trim and normalize company names
```sql
UPDATE layoffs_staging2
SET company = TRIM(company);
```

Clean inconsistent location entries
```sql
-- Remove ",Non-U.S."
UPDATE layoffs_staging2
SET location = LEFT(location, CHAR_LENGTH(location) - CHAR_LENGTH(',Non-U.S.'))
WHERE location LIKE '%,Non-U.S.';

-- Fix specific entries
UPDATE layoffs_staging2
SET location = 'Melbourne'
WHERE location = 'Melbourne,Victoria';

UPDATE layoffs_staging2
SET location = 'New York City'
WHERE location = 'New Delhi,New York City';

UPDATE layoffs_staging2
SET location = 'Hangzhou'
WHERE location = 'Non-U.S.' AND company = 'WeDoctor';

UPDATE layoffs_staging2
SET location = 'Mahé'
WHERE location = 'Non-U.S.' AND company = 'BitMEX';
```

Convert date and date_added to proper date format
```sql
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

UPDATE layoffs_staging2
SET date_added = STR_TO_DATE(date_added, '%m/%d/%Y');

ALTER TABLE layoffs_staging2
MODIFY COLUMN date_added DATE;
```

Handle missing/empty values in industry
```sql
UPDATE layoffs_staging2
SET industry = 'Software'
WHERE industry = '';
```

Dropping column row_num
```sql
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```

Convert percentage_laid_off to integer
```sql
UPDATE layoffs_staging2
SET percentage_laid_off = LEFT(percentage_laid_off, CHAR_LENGTH(percentage_laid_off) - 1);

UPDATE layoffs_staging2
SET percentage_laid_off = NULL
WHERE percentage_laid_off = '';

ALTER TABLE layoffs_staging2
MODIFY COLUMN percentage_laid_off INT;
```

Convert funds_raised to integer
```sql
UPDATE layoffs_staging2
SET funds_raised = RIGHT(funds_raised, CHAR_LENGTH(funds_raised) - 1);

UPDATE layoffs_staging2
SET funds_raised = NULL
WHERE funds_raised = '';

ALTER TABLE layoffs_staging2
MODIFY COLUMN funds_raised INT;
```

### 3) Exploratory Data Analysis (EDA)
Total layoffs insights
```sql
SELECT MAX(total_laid_off) FROM layoffs_staging2;

SELECT MAX(percentage_laid_off),  MIN(percentage_laid_off)
FROM layoffs_staging2
WHERE percentage_laid_off IS NOT NULL;
```

Top companies with highest layoffs
```sql
SELECT company, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC
LIMIT 10;
```

Top affected locations and countries
```sql
SELECT location, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY location
ORDER BY 2 DESC
LIMIT 10;

SELECT country, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY country
ORDER BY 2 DESC;
```

Year-wise trend
```sql
SELECT YEAR(date), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY YEAR(date)
ORDER BY 1 ASC;
```

Industry and stage impact
```sql
SELECT industry, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY industry
ORDER BY 2 DESC;


SELECT stage, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY stage
ORDER BY 2 DESC;
```

Top 3 companies with most layoffs by year
```sql
WITH Company_Year AS 
(
  SELECT company, YEAR(date) AS years, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY company, YEAR(date)
), 
Company_Year_Rank AS (
  SELECT company, years, total_laid_off, DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
  FROM Company_Year
)

SELECT company, years, total_laid_off, ranking
FROM Company_Year_Rank
WHERE ranking <= 3
AND years IS NOT NULL
ORDER BY years ASC, total_laid_off DESC;
```

Monthly rolling layoffs trend
```sql
WITH DATE_CTE AS 
(
  SELECT SUBSTRING(date,1,7) AS dates, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY dates
  ORDER BY dates ASC
)
SELECT dates, SUM(total_laid_off) OVER (ORDER BY dates ASC) AS rolling_total_layoffs
FROM DATE_CTE
ORDER BY dates ASC;
```

### Notes

- The dataset has several inconsistencies that were resolved using manual data cleaning.
- The rolling layoffs trend helps visualize the cumulative monthly impact.
- You can easily extend this project by creating dashboards using tools like Power BI, Tableau, or integrating with Python libraries.

### Contact

For questions or suggestions, feel free to open an issue or fork this repo and contribute.
