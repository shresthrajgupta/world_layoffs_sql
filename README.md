# Data Analysis using World Layoffs SQL Project

## Project Overview
This project uses the Layoffs 2022 dataset from Kaggle to analyze global tech layoffs using MySQL. The goal is to clean, transform, and perform exploratory data analysis (EDA) to extract meaningful insights from layoff trends across different companies, industries, and geographies.

*Dataset*: https://www.kaggle.com/datasets/swaptr/layoffs-2022

### Database Setup
```sql
CREATE DATABASE world_layoffs;
USE world_layoffs;

-- Import the dataset into a table named 'layoffs'

SELECT * FROM layoffs;
```

### Data Cleaning & Transformation
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

### Exploratory Data Analysis (EDA)
